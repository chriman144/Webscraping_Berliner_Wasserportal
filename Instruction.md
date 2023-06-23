# Webscraping open data from "Wasserportal Berlin"

The amount of data provided by public authorities increases constantly. The applications gaining these data sometimes still lack according to usability and flexibility. For Berlin a so called 'Portal' for general data about the current state of ground water and surface water was created. In general, a good job was done in here. You can access data of all known measuring points by a map, check hydrographs, download data of single stations and even can watch changes in time on a map view as animation. 

You can check it out in here: [Wasserportal Berlin](https://wasserportal.berlin.de/start.php)

![Screenshot of the website of open data for water in berlin](/Images/Screenshot_Wasserportal.png)

But let's pretend you want to get all the current data in a more flexible format and all historic data for your project about Berlin water resources. The documented API for the data is simply the adress of the website. So you need to load all your data manually or use some web scraping techniques. In here, I provide a Jupyter notebook with a small recipe downloading current ground water level data and all know historic data hosted by 'Wasserportal Berlin'.
You can download the Jupyter notebook (as of June 23 2023) and edit for your needs.

Desclaimer: Due to the disclaimer of the provider all data is unautited raw data.


<br/>

## Check robots.txt

Before starting our small adventure through waterworld of Berlin we have to check if there are an restrictions of scraping the provided data. Therefore, we check for some robots.txt, which usually contain all information needed.
First we check www.wasserportal.berlin.de/robots.txt, what leads us to a "Not found"-error. So let's check [berlin.de/robots.txt](www.berlin.de/robots.txt)

Ah here we go.

This says
> Any automated program to retrieve information from our public websites is only allowed subject to the following restrictions: 
>* It must clearly and uniquely identify itself in the HTTP User-Agent header.
>   https://webmasters.stackexchange.com/questions/6205/what-user-agent-should-i-set

And neither the website we want to scrape nor our used agent (selenium) is named (for the specific website). So it should be fine do the next steps.

__Another Disclaimer: This is not an legal advice. I assume no liability for the usage of scraping methods on the named website.__

## Preparing & importing necessary liberaries & functions

For this recipe we will use standard libraries for webscraping (*requests*, *BeautifulSoup*), data handling (*pandas*), time management (*time*) and for the more tricky part *selenium* as crawler and scraper.

I will not get in detail for the libraries. For more specific knowledge and background information read the docs.

```
# import necessary libaries/functions first
# for web scraping in general
import requests                                         
from bs4 import BeautifulSoup                           
import pandas as pd                                     
import time                                             

# selenium to scrape more nested data
from selenium import webdriver                          
from selenium.webdriver.common.by import By             
from selenium.webdriver.common.keys import Keys         
from selenium.webdriver.chrome.options import Options   
from selenium.webdriver.chrome.service import Service   
```
<br/> 

## Load current data
In the next step we define a function going all the steps to load the current data and store it in a pandas dataframe. In this example we will load the ground water data, but by adjusting the URL you can download almost every provided data. First let's check out the website visually.

Click [Current ground water level](https://wasserportal.berlin.de/messwerte.php?anzeige=tabelle&thema=gws)


![Screenshot of current ground water level](/Images/Screenshot_ground_water_level.png)

You can see that there is a somehow formated table. Chances are good, we can acces this later directly via a tag.

Let us check this by right clicking on the table and selecting element information. Here we can investigate the HTML-code of the website and lucky us. The table is a proper formated table with an own id-tag "pegeltab".

![Screenshot of element information table](/Images/Screenshot_element_information_table.png)


So let's start loading the webpage in our environment and extract the best out of it. In here, we first extract the table in it's format as HTML.

```
# url of current data
url = 'https://wasserportal.berlin.de/messwerte.php?anzeige=tabelle&thema=gws&nstoffid=10'  # change url if you want to scrape another parameter (e.g. surface water level)
    
# Create object page to load html
page = requests.get(url)

# use parser-lxml Change html to Python friendly format
# Get page's information
soup = BeautifulSoup(page.text, 'lxml')
raw_table = soup.find("table", id="pegeltab")
raw_table
```
<br/>
From this we first create a object for the header of our future dataframe and fill it with the actual column names. Because of some nasty formatting issues we have to tidy them up a bit afterwards. Subsequently, we create a pandas dataframe with these headers as column names and fill it row wise with the data from the scraped raw table (HTML table)

```
# Obtain every title of columns with tag <th>
headers = []
for i in raw_table.find_all("th"):
    title = i.text
    headers.append(title);

# Clean headers
headers = [x.replace('-', '') for x in headers]
headers = [x.replace('  ', ' ') for x in headers]

# Create a dataframe for data
df_current_data = pd.DataFrame(columns = headers)

# Create a for loop to fill df_current_data
for j in raw_table.find_all("tr")[1:]:
    row_data = j.find_all("td")
    row = [i.text for i in row_data]
    length = len(df_current_data)
    df_current_data.loc[length] = row
```
Now you have all the current surface water level data with coordinates (in EPSG 25833), date of measurement and some extras.

* Messstellennummer = station ID
* Bezirk = district
* Ausprägung = characteristics (meaning which data is collect by station)
* Grundwasserleiter = aquifer
* Grundwasserspannung = character whether the aquifer ist confined or not
* Datum = date
* Grundwasserstand = ground water level (m NHN)
* Flurabstand = depth to groundwater level
* Ganglinie = hydrograph
* Klissifikation = classification whether the ground water level is in high, mid or low state

The hydrograph column contains a link to further information as a small image and consequently seems empty in our dataframe, but without beeing NULL oder NaN. That's confusing. Therefore, we drop it before continuing.

```
df_current_data_clean = df_current_data.drop(columns = ['Ganglinie']) 
```

Maybe you have to change 'Ganglinie' to 'Ganglinien' depending on table you want to scrape.

<br/>

## Downloading historic data
If you want to load historic data of a station we have to use another tool *selenium*. This is caused by the fact, that the data is just accessible via a download button we have to click seperately for every station.
Selenium will run the browser of our choices to imitate clicking the necessary buttons to download the data. Depending on our browser we have to download some extra software (Webdriver, see below in chapter 'Scraüe historic data').

![Screenshot of page to download historic data](/Images/Screenshot_Download_Station_data.png)

Checking the URL for several stations we recognize that they are generated by a general path plus their station ID. So we can easily run through them all with a simple sequence and do the magic with selenium.

<br/>

### Extracting stations
This seems simple but still need to be done. We extract the station IDs first.

```
stations = df_current_data_clean['Messstellennummer']
```

<br/>

### Scrape historic data
In here, I use the the webdriver for Chrome (chromedriver), which I downloaded before a stored it in my favorite path. The chromdriver must match to your installed version of Chrome. Alternatively, you can find other webdrivers for instance for Firefox or Microsoft Edge. Check [Download page Selenium](https://pypi.org/project/selenium/) for more information.

In the beginning, we set some simple options to operate the browser via selenium headless (without seeing the operations on screen) or not, and defining our prefered path to download the data. Afterwards, we start the driver with these options.

```
#Set some seleniun chrome options
chromeOptions = Options()
chromeOptions.headless = False # change to True if you do not want to see the steps done on screen

# define path where to save the downloaded files
prefs = {"download.default_directory" : "/your/favorite/path"}
chromeOptions.add_experimental_option("prefs",prefs)

# create driver object
driver = webdriver.Chrome(executable_path='/path/to/your/chromdriver/chromedriver.exe', options=chromeOptions)
```

Now let's define a function for the repeated click work we can call later for the different URL of every station. In here, we use the XPATH of the asked buttos by investigating the specific elements, copying their XPATH and "clicking" it via the function click by selenium. In between the clicks we simply add some seconds of inactivity to avoid annoying the admin of the scraped website and overload the webpage.

```
# define function to navigate through website and click the download button
# if you want to download another data, you have to check the XPATH of the buttons
def download_via_browser(Download_URL):

    driver.implicitly_wait(10) # implicit wait until the asked features is loaded
    time.sleep(2) # 2 seconds waiting before download to avoid annoying the host admin
    driver.get(Download_URL)
    print ("starting Driver")
    button_1 = driver.find_element(by = By.XPATH, value ="/html/body/div[2]/div/div/div/div[4]/div[2]/a[2]") # find button to download section
    time.sleep(2) # seconds
    button_1.click()
    button_2 = driver.find_element(by = By.XPATH, value ='//*[@id="form_gw_c"]/button') #find download button
    time.sleep(2) # seconds
    button_2.click()
```

![Screenshot copying XPATH](/Images/Screenshot_XPATH.png)

Now we just have to run this function with generated URLs 

```
for m in stations.index:
        url_no = str(stations[m])
        # print(url_no)
        Download_URL = ("https://wasserportal.berlin.de/station.php?anzeige=d&thema=gws&station="+str(url_no))
        download_via_browser(Download_URL)
```

Depending on your set of stations this can take some time. Maybe you go to drink a coffee or something. 

:coffee: