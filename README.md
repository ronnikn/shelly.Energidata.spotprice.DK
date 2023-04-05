 
# Turn Shelly On/Off Based on Energi Data Service Danish current Spotprice
 
This code pulls the current price from energidataservice API

# Setup

- ### First enter The Shelly's Web UI By entering it's ip-adress in a browser.

![My Image](Screenshots/bar.png)
 

- ### Go to Setting

![My Image](Screenshots/1.png)


- ### Go to Debug

![My Image](Screenshots/2.png)


- ### Enable Websocket debug and Save Settings

![My Image](Screenshots/3.png)


- ### Add Script

![My Image](Screenshots/4.png)


- ### Paste in the Code (also found in spotprice_dkk.js)

```
 let CONFIG = {
  api_endpoint: "http://api.energidataservice.dk/dataset/Elspotprices?filter={%22PriceArea%22:[%22LANDEKODE%22]}&columns=SpotPriceDKK,HourDK&sort=HourDK&start=now-P1D&limit=2&offset=23",
  switchId: 0,             // ID of the switch to control
  price_limit: 2000,        // EUR/MWh. Vat not included
  update_time: 60000,      // 1 minute. Price update interval in milliseconds
  reverse_switching: false // If true, switch will be turned on when price is over the limit
};

let current_price = null;
let last_hour = null;
let last_price = null;
let state = null;

function getCurrentPrice() {
  Shelly.call(
    "http.get",
    {
      url: CONFIG.api_endpoint,
    },
    function (response, error_code, error_message) {
      if (error_code !== 0) {
        print(error_message);
        return;
      }
      current_price = JSON.parse(response.body).records[0]["SpotPriceDKK"];
      print("Updated current price!");
    }
  );
}

function changeSwitchState(state) {
  let state = state;

  if(state === false) {
    print("Switching off!");
  } else if(state === true) {
    print("Switching on!");
  } else {
    print("Unknown state");
  }

  Shelly.call(
    "Switch.Set",
    {
      id: CONFIG.switchId,
      on: state,
    },
    function (response, error_code, error_message) {
      if (error_code !== 0) {
        print(error_message);
        return;
      }
    }
  );
}

Timer.set(CONFIG.update_time, true, function (userdata) {
  Shelly.call("Sys.GetStatus", {}, function (resp, error_code, error_message) {
    if (error_code !== 0) {
      print(error_message);
      return;
    } else {
      let hour = resp.time[0] + resp.time[1];
      //update prices
      if (last_hour !== hour) {
        print("update hour");
        last_hour = hour;
        getCurrentPrice();
      }

      //check if current price is set
      if (current_price !== null) {

        //Normal switching. Turn relay off if price is over the limit
        if(CONFIG.reverse_switching === false) {
          if (current_price >= CONFIG.price_limit) {
            //swith relay off if price is higher than limit
            changeSwitchState(false);
          } else {
            //swith relay on if price is lower than limit
            changeSwitchState(true);
          }
        }

        //Reverse switching. Turn relay on if price is over the limit
        if(CONFIG.reverse_switching === true) {
          if (current_price >= CONFIG.price_limit) {
            //swith relay on if price is higher than limit
            changeSwitchState(true);
          } else {
            //swith relay off if price is lower than limit
            changeSwitchState(false);
          }
        }

      } else {
        print("Current price is null. Waiting for price update!");
      }
      

      print(current_price);
    }
  });
});
```
## Configure API endpoint
Find `api_endpoint` and change `LANDEKODE` to 
- `DK1` for Sjælland 
- `DK2` For Fyn / Jylland

EX. For Sjælland
 `api_endpoint: "http://api.energidataservice.dk/dataset/Elspotprices?filter={%22PriceArea%22:[%22DK1%22]}&columns=SpotPriceDKK,HourDK&sort=HourDK&start=now-P1D&limit=2&offset=23"`
 
## Set your price point  
Find configuration value `price_limit` and change value for when your device turns on or off. Prices don’t include VAT and are measured in DKK/MWh
### Example
```  price_limit: 1500 ```
Will set toggling threshold for the device to 1.5 DKK/kWh
 
## Setting relay to switch.
Most shelly devices have only one output(relay). If you want to change the output channel find `switchId` and set it to the desired output.
 


