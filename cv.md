# Evgeniia Sakhnova

![](/photo.png)

## Python developer

### Contacts
**Phone:** [+7(920)4213354]()<br>
**Email:** [zhenyazav6666@gmail.com]()<br>
**Telegram:** [@Jane_Prince](https://t.me/Jane_Prince)<br>
**GitHub:** [JanePrince666](https://github.com/JanePrince666)<br>
[LinkedIn](http://www.linkedin.com/in/evgenia-sakhnova-51795624b)
### Personal Information
**Date of Birth:** 31.05.1996<br>
**City:** Tbilisi
### Summary
My thirst for knowledge is strong. I strive for more than just a surface-level understanding.
I want to grasp the intricacies and nuances. By actively exploring Python's vast ecosystem,
I keep my skills sharp and my mind adaptable, preparing myself to tackle any challenges that come my way.<br>
Additionally, I have experience creating my own projects and educational projects,
as well as completing a 3-month internship that further enhanced my practical skills.
### Skills
- Python
- OOP
- SQL
- MySQL
- Git
- Docker
- Aiogram 3
- Django
- Analytical skills
### Code example
```python
import re
import time

import requests
import multiprocessing
from bs4 import BeautifulSoup as bs
from tor_python_easy.tor_control_port_client import TorControlPortClient

from config import DATA_FOR_DATABASE
from db_management_OOP import ParsingChannels, PostingList, MonitoredTelegramChannels


tor_control_port_client = TorControlPortClient('tor', 9050)

# Establish a connection with Tor Control Port and change the IP address through Tor every 5 seconds.
tor_control_port_client.change_connection_ip(seconds_wait=5)

# Set the proxy server configuration to use SOCKS5 proxy on localhost and
# port 9050 for HTTP and HTTPS requests.
proxy_config = {
    'http': 'socks5://tor:9050',
    'https': 'socks5://tor:9050',
}


def get_html(url: str) -> bs:
    """
    Return BeautifulSoup object by url

    :param url: URL of the web page
    :type url: str
    :return: BeautifulSoup object
    """
    try:
        response = requests.get(url, proxies=proxy_config)
        soup = bs(response.content, 'html.parser')
        return soup
    except Exception as e:
        print(f"Failed to get HTML: {e}")


def check_on_stub(url: str) -> bs | bool:
    """
    Check whether the received page is a Telegram stub

    :param url: URL of the web page
    :type url: str
    :return: BeautifulSoup object if the page is not a stub, False otherwise
    """
    soup = get_html(url)
    try:
        if soup.find('div', class_="tgme_page_additional"):
            return False
        else:
            return soup
    except:
        return False


def get_posts(url: str) -> list | bool:
    """
    Return posts found on the channel page as a list

    :param url: URL of the channel page
    :type url: str
    :return: List of post elements if found, False otherwise
    """
    soup = check_on_stub(url)
    if soup:
        posts = soup.find_all('div', class_="tgme_widget_message text_not_supported_wrap js-widget_message")
        return posts
    else:
        return False


def pars_channel(url: str, last_post_number: int, first_launch: bool):
    """
    Parse Telegram channel. The result is reflected in the database

    :param url: URL of the channel page
    :param last_post_number: Number of the last parsed post
    :param first_launch: Flag indicating the first launch of the bot
    """
    # Extract posts from the channel using the get_posts function.
    posts = get_posts(url)
    attempt_counter = 0

    # Retry getting posts, changing the IP address through Tor on unsuccessful attempts.
    while not posts:
        attempt_counter += 1
        tor_control_port_client.change_connection_ip(seconds_wait=5)
        posts = get_posts(url)
        if attempt_counter > 5:
            print("Error connecting to Tor")
    last_post_number = last_post_number

    # Check each post for the presence of text information
    for post in posts:
        post_url_data = post['data-post']
        post_number = int(re.search('\/\d+', post_url_data).group()[1:])
        post_url = "https://t.me/" + post_url_data
        try:
            text = post.find_all('div', class_="tgme_widget_message_text js-message_text")[0].text
        except:
            text = ""

        # When a new post is detected, add it to the posting list for channel subscribers,
        # if this is not the first bot launch and if there are channels subscribed to this Telegram channel
        if post_number > last_post_number:
            last_post_number = post_number
            subscribed_user_channels = MonitoredTelegramChannels(
                *DATA_FOR_DATABASE).get_user_channels_subscribed_on_tg_channel(url)
            if not first_launch and len(subscribed_user_channels) > 0:
                for user_channel in subscribed_user_channels:
                    PostingList(*DATA_FOR_DATABASE).add_to_posting_list(post_url, text, user_channel)

    # Update the last post number for the channel in the database
    ParsingChannels(*DATA_FOR_DATABASE).change_channel_last_post(url, last_post_number)


def get_channel_list() -> list[str]:
    """
    Return a generator for get_new_posts

    :return: List of channel URLs
    """
    connection = ParsingChannels(*DATA_FOR_DATABASE)
    channel_list = connection.get_channels_list()

    # Split the channel list into blocks of 20 channels each and yield these blocks using a generator
    for i in range(0, len(channel_list), 20):
        unit = channel_list[i:i + 20]
        yield unit


def get_new_posts():
    """
    Endlessly parse Telegram channels in blocks of 20 at a time
    """
    first_launch = True  # Set the first launch flag to True.

    # Infinite loop for processing new posts.
    while True:
        channels = get_channel_list()
        for unit in channels:

            # For each of the 20 channels, start a parallel parsing process
            for url, start_post in unit:
                t = multiprocessing.Process(target=pars_channel, args=(url, start_post, first_launch,))
                t.start()

            # After processing 20 channels, wait 10 seconds to avoid starting an infinite number of
            # processes simultaneously. Then continue.
            time.sleep(10)

        # After completing the channel parsing for the first time, set the first launch flag to False.
        first_launch = False
```
