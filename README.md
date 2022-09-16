# Arduino AM2303

AM2303 is an Arduino-compatible thermometer (temperature sensor) and hygrometer
(humidity sensor).
It is a wired version of the DHT22 sensor.

I am setting up a bunch around my house to monitor temperature and humidity in
it as well as in the barn and my shop.
I might throw in a couple more in the future to monitor my shed as well.

I am using the sensors to collect the temperature and humidity data.
An Arduino is used to communicate with the sensor.
The Arduino program follows.
The `DHT.h` library is installed using Arduino IDE.
Go to Tools > Manage Libraries > search for "DHT sensor library" by Adafruit
(at the time of writing, version 1.4.4) and install it.
More about the library here: https://github.com/adafruit/DHT-sensor-library

The Arduino program prints temperature and humidity values to the serial output
of the Arduino, which can be monitored using the Arduino IDE.
Use Tools > Serial Monitor for that.

```ino
// Add via Tools > Manage Libraries > "DHT sensor library"@1.4.4 by Adafruit
// See https://github.com/adafruit/DHT-sensor-library
#include <DHT.h>

#define DHTPIN 2
#define DHTTYPE DHT22 // Note that AM2302 is the wired version of DHT22

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(9600);
  dht.begin();
}

// Monitor via Tools > Serial Monitor
void loop() {
  Serial.println("Temperature: " + String(dht.readTemperature()) + " °C");
  Serial.println("Humidity: " + String(dht.readHumidity()) + " %");
  
  // Wait for two seconds - the temporal resolution of the sensor
  delay(2000);
}
```

- [ ] Pull this out to its own file in the repository later
  
  Right now I'm just collecting all the scraps in one place, but this will be
  useful later to be able to open it in the Arduino IDE easily.

The serial output when deployed, not just debugging, is instead monitored using
a PowerShell script on an Intel Nuc computer that's always-on at the house.

- [ ] Attach the PowerShell script source here and clone it on the Nuc

The script reads out the serial output of the Arduino and sends it to a Deno
Deploy edge function.

The code for the Deno function follows and the function lives on this URL:
https://sensor.deno.dev

This is the dashboard link for it: https://dash.deno.com/projects/sensor

```typescript
import { serve } from "https://deno.land/std@0.140.0/http/server.ts";
import * as postgres from "https://deno.land/x/postgres@v0.14.0/mod.ts";

const pool = new postgres.Pool(Deno.env.get("CONNECTION_STRING")!, 3, true);

serve(async request => {
  const connection = await pool.connect();
  const { password, line, stamp } = await request.json().catch(() => null);
  if (password !== Deno.env.get("PASSWORD")) {
    throw new Error('Bad password');
  }

  try {
    let value;
    switch (line.replace(/\d/g, '#')) {
      case 'Humidity: ##.## %\r': {
        value = Number(line.slice('Humidity: '.length, 'Humidity: ##.##'.length));
        await connection.queryObject`INSERT INTO humidity (value, happened_at) VALUES (${value}, ${stamp})`;
        break;
      }
      case 'Temperature: ##.## ??C\r': {
        value = Number(line.slice('Temperature: '.length, 'Temperature: ##.##'.length));
        await connection.queryObject`INSERT INTO temperature (value, happened_at) VALUES (${value}, ${stamp})`;
        break;
      }
    }

    console.log({ line, value });

    return new Response("OK", {
      headers: { "content-type": "text/plain" },
    });
  }
  catch (error) {
    return new Response("Hello World!", {
      headers: { "content-type": "text/plain" },
    });
  }
  finally {
    connection.release();
  }
});
```
 
The function collects the calls, checks them for a secret password to make sure
no one can flood the endpoint with junk data and stores the data in Supabase by
directly writing to the backing Postgres instance.

It also prints out the data into the console so the function logs can be viewed
in Deno Deploy easily.

- [ ] Get rid of Postgres connection pooling here
  
  I copied this off a tutorial and have not tweaked it to my liking just yet.

- [ ] Figure out the VS Code setup for Deno imports to avoid design time errors

  I think it might be enough to install the Deno VS Code extension but not sure

I am vieweing the data using an iOS application called Charts: for Supabase.
https://apps.apple.com/us/app/charts-for-supabase/id1612680145

I am also generating charts using Plotly. I have code to connect to the database
using `psql`, generating a CSV and a JS data module and displaying the charts
based on that, but it is not in this repository yet.

- [ ] Merge the untracked bits on my Desktop (`sensor` directory) to this repo

This system is currently completely passive, if there were to be a temperature
or humidity swing (fire or flood [e.g.: due to a burst pipe]), I'd only find out
retrospectively.

I plan on adding outlier detection which would run on each call to the Deno
function, compare to a window of most recent values and see if the rate of
change is too high.
I might integrate with Twilio or use the of the number of iOS applications that
offer a client and a service to call to generate a notification on the phone.

- [ ] Break down active change detection and notification implementation
