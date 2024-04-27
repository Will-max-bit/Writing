# Monitoring Blog

Upon starting my current role, I was immediately confronted with a challenging situation: essential data needed for actionable decisions on our equipment was spread across five different sources. Each source not only required unique authentication but also had its data buried deep within, complicating the process of data correlation which could only be done manually or by arranging multiple desktop windows effectively.

My web development background prompted an initial solution to consolidate monitoring for every IoT device into one server interface, displayed via the DOM using ChartJs. Although this approach seemed feasible for small-scale operations, it was too resource-intensive and akin to reinventing the wheel in 2024. Further complications arose with IoT devices located on the network's edge where connectivity is unreliable due to weather conditions and other site-specific variables. These often left sites unresponsive and unqueryable. My initial assignment was to integrate our IoT devices into a Nagios system. While Nagios excelled as a robust alerting system and could be made highly flexible with some Python enhancements, it fell short in handling metrics—a crucial component for predictive analytics and proactive condition-based alerting.

This led me to adopt Prometheus for data collection and Grafana for visualization, which I'll discuss here using a slightly redacted version of my Python exporter.

# The Challenge

I manage roughly 25 sites, each with variations in power systems (DC or AC), equipment, and configurations due to staggered construction schedules. For instance, a typical DC site would have five distinct devices: a camera, radios, a CBW, a Solar Controller, and a router. My stint with Nagios had provided a foundation in SNMP and how to connect with edge devices, which I initially thought was sufficient.

I began by crafting individual functions for each type of device interaction, which were tested successfully in isolation and then amalgamated into a main method with a sleep function to regulate Prometheus's looping mechanism. While this setup functioned, I soon realized it wouldn't scale effectively due to the burgeoning size of my metrics dictionary.

# The Solution

To streamline operations, I structured the configuration into three scripts:
    1. Site Configuration: The first script organizes all sites as a dictionary of dictionaries, allowing for simplified data iteration and dynamic variable naming. For example, the entry for site 'Pine' might look like:

```python
"Site_A": {
    "Device_A": "192.168.1.10",
    "device_B_Scrape": "192.168.2.20",
    "Device_C": "192.168.3.30",
}
```

This structure facilitates efficient metric generation by dynamically correlating the site and device names to create unique metric identifiers, such as Pine_Solar_Controller_load_voltage.

    2. Metric Definitions: The second script defines metrics to ensure the dynamically generated names during the data collection phase correspond accurately to predefined metrics. This was an area where I later realized the benefit of labels, which would have greatly enhanced our ability to filter data and create dynamic dashboards.


```python
"SiteSensor_battery_voltage": Gauge("site_device_a_battery_voltage", "SiteSensor Battery Voltage"),
"SiteSensor_supply_voltage": Gauge("site_device_a_supply_voltage", "SiteSensor Supply Voltage"),
"SiteSensor_box_temp": Gauge("site_device_a_box_temp", "SiteSensor Box Temperature"),
"SiteSensor_box_humidity": Gauge("site_device_a_box_humidity", "SiteSensor Box Humidity"),

"SiteSensor_array_current": Gauge("site_device_b_scrape_array_current", "SiteSensor Array Current"),
"SiteSensor_array_voltage": Gauge("site_device_b_scrape_array_voltage", "SiteSensor Array Voltage"),
"SiteSensor_charge_current": Gauge("site_device_b_scrape_charge_current", "SiteSensor Charge Current"),
"SiteSensor_sweep_voc": Gauge("site_device_b_scrape_sweep_voc", "SiteSensor Sweep VOC"),
"SiteSensor_charge_ah": Gauge("site_device_b_scrape_charge_ah", "SiteSensor Charge AH"),
"SiteSensor_sweep_vmp": Gauge("site_device_b_scrape_sweep_vmp", "SiteSensor Sweep VMP"),
"SiteSensor_charge_power": Gauge("site_device_b_scrape_charge_power", "SiteSensor Charge Power"),
"SiteSensor_sweep_pmax": Gauge("site_device_b_scrape_sweep_pmax", "SiteSensor Sweep PMAX"),
"SiteSensor_charge_state": Gauge("site_device_b_scrape_charge_state", "SiteSensor Charge State"),
"SiteSensor_gen_battery_voltage": Gauge("site_device_b_scrape_battery_voltage", "SiteSensor Gen Battery Voltage"),
"SiteSensor_load_state": Gauge("site_device_b_scrape_load_state", "SiteSensor Load State"),
"SiteSensor_battery_current": Gauge("site_device_b_scrape_battery_current", "SiteSensor Battery Current"),
"SiteSensor_load_current": Gauge("site_device_b_scrape_load_current", "SiteSensor Load Current"),
"SiteSensor_target_regulation": Gauge("site_device_b_scrape_target_regulation", "SiteSensor Target Regulation"),
"SiteSensor_heat_sink": Gauge("site_device_b_scrape_heat_sink", "SiteSensor Heat Sink"),
"SiteSensor_battery_temp": Gauge("site_device_b_scrape_battery_temp", "SiteSensor Battery Temperature"),

"SiteSensor_tx": Gauge("site_sensor_tx", "SiteSensor TX"),
"SiteSensor_rx": Gauge("site_sensor_rx", "SiteSensor RX"),
"SiteSensor_tx_EIRP": Gauge("site_sensor_tx_EIRP", "SiteSensor TX EIRP"),
"SiteSensor_antenna_gain": Gauge("site_sensor_antenna_gain", "SiteSensor Antenna Gain"),
"SiteSensor_cable_loss": Gauge("site_sensor_cable_loss", "SiteSensor Cable Loss"),
"SiteSensor_tx_rate": Gauge("site_sensor_tx_rate", "SiteSensor TX Rate"),
"SiteSensor_tx_bytes": Gauge("site_sensor_tx_bytes", "SiteSensor TX Bytes"),
"SiteSensor_tx_pps": Gauge("site_sensor_tx_pps", "SiteSensor TX PPS"),
"SiteSensor_rx_bytes": Gauge("site_sensor_rx_bytes", "SiteSensor RX Bytes"),
"SiteSensor_rx_pps": Gauge("site_sensor_rx_pps", "SiteSensor RX PPS"),
"SiteSensor_connected": Gauge("site_sensor_connected", "SiteSensor Connected"),
```
    3. Data Collection Engine: The third script serves as the backbone of our monitoring operation. It performs multiple functions across each station, including web scraping for data collection, API interactions, SNMP data retrieval, error handling, and ultimately serving the collected metrics on port 8000.

# The Execution

The main loop of our data collection script iterates through each site, retrieving the name and IP address to pass to device-specific functions. This allows for dynamic metric generation and ensures our monitoring system remains responsive even when some sites are temporarily down. Here's a glimpse of how this logic is executed:

```python
    async def main():
    for site in SITES:
        for device in SITES[site]:
            if device == "Radio":
                await radio_snmp(SITES[site][device], site)
            elif device == "Solar":
                await emc_snmp(SITES[site][device], site)
            elif device == "CBW":
                await cbw(SITES[site][device], site)
            elif device == "Genstar":
                await genstar_scrape(SITES[site][device], site)
            else:
                logger = logging.getLogger(__name__)
                logger.error(f"Error with {site}")
    await asyncio.sleep(60)
```

By modularizing our approach and using advanced data collection techniques, we've not only streamlined our monitoring processes but also enhanced our capability for predictive maintenance and operational efficiency. This setup exemplifies how modern monitoring systems can be both robust and agile, adapting to the complexities of managing diverse and geographically dispersed infrastructure.

## Functions

#### WEBSCRAPER

To maintain separation of concerns, this function initiates the web scraping process, starting the request and collecting the relevant data into two lists, each labeled with the current station's name. One significant lesson learned during the development of this function was the importance of using finally. The device webpages we target are situated on remote mountain locations, often 10 to 70 miles from the nearest wired Ethernet connection. This geographical challenge leads to highly unpredictable response times. Consequently, a 45-second timeout and a finally clause are crucial to prevent the program from hanging when a device is unresponsive. If the scrape is successful, the data is then passed to a helper function to ensure accuracy in the extracted values.

```python
async def genstar_scrape(url, site):
    driver = None
    try:
        options = Options()
        options.add_argument("--headless")
        driver = webdriver.Firefox(options=options)
        driver.get(f"http://{url}")
        wait = WebDriverWait(driver, 45)
        elements = wait.until(
            EC.presence_of_all_elements_located((By.CLASS_NAME, "tiles"))
        )
        element_one = elements[0].text.split("\n")
        element_two = elements[1].text.split("\n")
        output = scrape_helper(site, element_one, element_two)
        for key, value in output.items():
            await safe_set_metric(key, value)
    except Exception as err:
        logger = logging.getLogger(__name__)
        logger.error(f"Error with {site} - Error type: {type(err).__name__}, error: {err}")
    finally:
        if driver:
            driver.quit()
```

#### WEBSCRAPER PARSE

The parser function is meticulously designed with separation of concerns in mind. Upon invocation, it merges two lists into one, resulting in a structure like `'array current',84 V, 'charge current', 3.3 A, 'sweep voc', 45 ah, ....` It then uses slicing to extract every second element starting from the second position, transforming the list into `84 V, 3.3 A, 45 ah` Subsequently, I apply a regular expression to remove the unit designations from each number, yielding a streamlined list: `84, 3.3, 45`

```python
def scrape_helper(site, element_one, element_two):
    elements = element_one + element_two
    value = elements[1::2]
    metric_values = []
    for val in value:
        sanitized_value = re.search(r"-?\d*\.?\d+", val)
        if sanitized_value:
            metric_values.append(float(sanitized_value.group()))
        else:
            metric_values.append(None)
    metric_names = [
        "array_current",
        "array_voltage",
        "charge_current",
        "sweep_voc",
        "charge_ah",
        "sweep_vmp",
        "charge_power",
        "sweep_pmax",
        "charge_state",
        "gen_battery_voltage",
        "load_state",
        "battery_current",
        "load_current",
        "target_regulation",
        "heat_sink",
        "battery_temp",
    ]
    metric_names = [f"{site}_{name}" for name in metric_names]
    metrics = dict(zip(metric_names, metric_values))
    excluded_keys = {f"{site}_charge_state", f"{site}_load_state"}
    output = {key: value for key, value in metrics.items() if key not in excluded_keys}
    return output
```

##### SNMP

The SNMP function was relatively straightforward to develop because I had prior experience writing a program for Nagios that managed SNMP values from our cameras, radios, and solar controllers. The primary advancements in this version include the use of asynchronous programming techniques and a dedicated asynchronous SNMP library. While I will not delve deeply into the complexities of asynchronous programming here, it's crucial for handling the unpredictable response times and maintaining functionality when sites are down. Asynchronous programming inherently prevents the program from freezing during prolonged response times. By employing asynchronous methods and an SNMP library tailored for such tasks, I aimed to mitigate any issues arising from delayed or missing responses.

The function's logic operates as follows: it takes the current URL and site from the loop as parameters, sends an SNMP request to the device, and retrieves a list of desired OIDs from the dictionary specified earlier. The asynchronous library awaits a response, and upon receipt, an inner loop iterates over the OID keys and the corresponding values returned from the device. If a key/value pair contains a valid (non-null) value, that metric name and its corresponding value are passed to a method designed for safe setting and further parsing. Any issues encountered during this process are logged for debugging.

```python
async def emc_snmp(url, site):
    url = url
    try:
        async with aiosnmp.Snmp(
            host=url, port=000, community="----", timeout=10, retries=2
        ) as snmp:
            oids = {
                "load_voltage": "1.3.6.1.4.1.33333.5.32.0",
                "load_current": "1.3.6.1.4.1.33333.5.34.0",
                "array_voltage": "1.3.6.1.4.1.33333.5.31.0",
                "charge_current": "1.3.6.1.4.1.33333.5.33.0",
                "battery_voltage": "1.3.6.1.4.1.33333.5.30.0",
                "heat_sink": "1.3.6.1.4.1.33333.5.38.0",
                "battery_temp": "1.3.6.1.4.1.33333.5.39.0",
                "ambient_temp": "1.3.6.1.4.1.33333.5.40.0",
                "sweep_Pmax": ".1.3.6.1.4.1.33333.5.64.0",
            }
            results = await snmp.get(list(oids.values()))
            for var_name, res in zip(oids.keys(), results):
                if res.value:
                    value = float(res.value.decode("utf-8"))
                    await safe_set_metric(f"{site}_{var_name}", value)
    except Exception as err:
        logger = logging.getLogger(__name__)
        logger.error(
            f"Error with {site} - Error type: {type(err).__name__}, error: {err}"
        )
```
# Conclusion

And that concludes this blog post. I've also prepared a brief tutorial that uses Python to generate random metrics via various data collection methods. It demonstrates how to integrate Prometheus with your Python exporter and subsequently connect Grafana to Prometheus for data visualization. Thank you for reading, and I'm eager to explore further possibilities in metric collection.