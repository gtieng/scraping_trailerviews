# ETL - Scraping YouTube Video Playlist Stats with Python
One of the most challenging aspects of working in social media marketing for entertainment is getting an accurate portrayal of reach when a new video is released as part of a major beat.

A big part of your reach will come from owned channels, while others may come from media parters, and even superfans who rip video content for their own channels. Believe me when I say I've spent a large deal of time between YouTube and a manual spreadsheet trying to record the internet's reception of our campaign's content after the first day, second day, third day and so on...

In this project, I'll walk you through a webscraping tool I developed to automate YouTube statistics recording for loading into a dataframe or database.

## Importing Essential Libraries
```
##browser controller
from selenium import webdriver
import time

#webscraping and string cleaning
from bs4 import BeautifulSoup
from datetime import datetime, timedelta
import re

#dataframe management
import pandas as pd
```

## String Cleaning Functions
Before we fire up the browser controller, it's important to first walk through a few of these string cleaning functions that are unique to the YouTube platform.

![](https://camo.githubusercontent.com/d101c1c1d25165fbd6a5238a376617eb697e4754/68747470733a2f2f6769746875622e636f6d2f677469656e672f796f75747562655f747261696c65725f736372617065722f626c6f622f6d61737465722f6d61726b646f776e5f696d616765732f79745f6b6d2e706e673f7261773d74727565)

Upon inspection of the metadata we want to scrape, you'll see that subscribers, views, and likes/dislikes will result in strings with commas, Ks and Ms (for thousands and millions, respectively) that will all need to be cleaned and cast into a numeric type. The following code block is a function to do just that.

```
def km_cleaner(value):
    #removes subscribers or views from string
    value = re.sub(' subscribers', '', value)
    value = re.sub(' views', '', value)
    
    #casts string to thousand or million numeric based on K or M value
    if 'K' in value:
        value = float(re.sub('K', '', value))
        value *= 1000
        return int(value)
    elif 'M' in value:
        value = float(re.sub('M', '', value))
        value *= 1000000
        return int(value)
    else:
        return int(value)
```

The other string to format for this function is the date. In the occurence that the uploaded video is less than a day old, YouTube may display the date in hours from its premiere date as shown below:

![](https://camo.githubusercontent.com/01cca9fd271aa3205594f2397c00293f6a6df372/68747470733a2f2f6769746875622e636f6d2f677469656e672f796f75747562655f747261696c65725f736372617065722f626c6f622f6d61737465722f6d61726b646f776e5f696d616765732f79745f61676f2e706e673f7261773d74727565)

```
def date_uploaded(value):
    value = re.sub('•', '', value)
    if 'ago' in value:
        value = int(re.search(r'\d+', value).group())
        uploaded = (datetime.now() - timedelta(hours=value))
        return datetime.strftime(uploaded, '%b %-d, %Y')
    else:
        return value
```

## Web Scraping Video Pages
With the string cleaning functions complete, we are now able to scrape single YouTube video pages to collect its stats. Using BeautifulSoup, we'll be able to extract the following information for our dataframe:

- Video Title
- Video URL
- Account Name
- Subscribers
- Video Views
- Date Uploaded
- Number of Likes
- Number of Dislikes

```

def get_stats(youtube_url):
    
    #browser controller
    driver = webdriver.Chrome('resources/chromedriver')
    driver.get(youtube_url)
    time.sleep(3)
    soup = BeautifulSoup(driver.page_source)
    driver.quit()
    
    #values that need no formatting
    title_value = soup.find('h1').text
    account_value = soup.find('yt-formatted-string', class_='ytd-channel-name').text
    
    #values that need date_uploaded function
    date_value = date_uploaded(soup.find('div', id='date').text)
    
    #values that need km_cleaner function
    subs_value = km_cleaner(soup.find('yt-formatted-string', id='owner-sub-count').text)
    view_count = km_cleaner(soup.find('span', class_='short-view-count').text)
    likes_value = km_cleaner(soup.find_all('yt-formatted-string', class_='ytd-toggle-button-renderer')[0].text)
    dislikes_value = km_cleaner(soup.find_all('yt-formatted-string', class_='ytd-toggle-button-renderer')[1].text)
    
    #combined dictionary of values
    stats = {'date_uploaded': date_value,
             'account': account_value,
             'video_title': title_value,
             'views': view_count,
             'subscribers': subs_value,
             'likes': likes_value,
             'dislikes': dislikes_value,
             'url': youtube_url
            }
    
    return stats
```

## Web Scraping Playlist Pages
Now that we have the ability to scrape one page based on the provided URL, we'll now write a scraping function to create a list of extracted video urls from a YouTube playlist page. The hardest challenge of this step is navigating the infinite scroll to load the full list of videos when there is more than 100 videos in the playlist. This problem was solved by programming the scroller to go to the bottom of the page for every 100 videos listed in the vidieo count summary.

```
def get_video_urls(youtube_playlist_url):
    #webdriver setup & html scrape
    driver = webdriver.Chrome('resources/chromedriver')
    driver.get(youtube_playlist_url)
    soup = BeautifulSoup(driver.page_source)

    #isolate the video stats
    video_stats = soup.find_all('yt-formatted-string')[5].text
    #use regex to isolate video number
    no_of_videos = re.sub(',', '', video_stats)
    no_of_videos = re.sub(' videos', '', video_stats)
    #number of scroll based on 100 thumbnails per scroll
    no_of_scrolls = int(int(no_of_videos) / 100 + 1)

    #scroll page for every 100 videos
    for i in range(no_of_scrolls):
        driver.execute_script("window.scrollBy(0, 12000);")
        time.sleep(3)

    soup = BeautifulSoup(driver.page_source)
    driver.quit()

    #extract youtube playlist urls
    matches = soup.find_all('a', class_='yt-simple-endpoint style-scope ytd-playlist-video-renderer')
    youtube_urls = []
    for url in matches:
        string = url.get('href')
        substring = re.search('\/watch\?v=([^&]+)', string).group()
        youtube_urls.append('http://www.youtube.com' + substring)    
    return youtube_urls
```
Finaly, we'll chain the two scraping functions into one.

```
def create_export(url):
    all_urls = get_video_urls(url)
    all_stats = []
    for video in all_urls:
        all_stats.append(get_stats(video))
    return all_stats
```
## Demonstration of the Tool
```
wwe = create_export('https://www.youtube.com/playlist?list=PL7qtZGedQPadDiw6Y-XgiQIk035CQc3pm')
df = pd.DataFrame(wwe)
df.sort_values("views", ascending=False)
```
![](https://github.com/gtieng/youtube_trailer_scraper/blob/master/markdown_images/yt_df.png?raw=true)

## Authors

**Gerard Tieng** — *Data Analyst & Social Media Marketer*
- [http://www.twitter.com/gerardtieng](http://www.twitter.com/gerardtieng)
- [http://www.linkedin.com/in/gerardtieng](http://www.linkedin.com/in/gerardtieng)
- [http://www.github.com/gtieng](http://www.github.com/gtieng)
