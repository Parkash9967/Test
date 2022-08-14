# MediaScraper

Collects and processes data required for Big Data analysis, saving the output in a .csv file in a structured format.

## Abstract
The main source of the data is the [hirkereso.hu](https://hirkereso.hu) webpage, which is where news content is collected from, as well other associated information, such as the appearance in the top 1 / 2 / 4 / 8 / 12 / 24 hours categories, article author and publication date, article text, etc. This information is then exported into a .csv file, which is then further processed.

## Tech info
The project is created in Python, specifically version 3.8, but can run on previous 3.x versions.

The basis of the project is the [Scrapy](https://docs.scrapy.org/en/latest/) framework, with a few extra additions. The process builds on the Scrapy "Spider" classes, which have been written in accordance to the documentation, but there are other classes and commands for text parsing, processing, and saving into files for example.

The project has its own virtual environment.

There are separate YAML configuration files so parameters can be modified and expanded as needed.

Everything is based on iterators/generators, so working with large amounts of data is not an issue and running out of memory is not a concern.

### Crawling workflow
1. Every hirkereso.hu "top" subpage is crawled and saved into each top category in a structured format to a .json file (temp/top_pages_output.json)
   
2. The outlet "boxes" on hirkereso.hu's home page are crawled (the ones to be excluded are predefined) and the outlet subpage URI-s are collected into a collection. Once that's complete, then these subpages are also opened and every news article in the "Friss" section are collected together with title and URL. At this point, each article gets a dummy variable for the corresponding top X hours categories if it was found in there or not (0 if not, 1 if yes). Then this step's output is dumped into a .json file (temp/main_output.json).

3. After the first two steps, main_output.json is processed: the data is collected from each outlet subpage (Origo, Index, etc.) and then each scraped URL content is then saved into a .json file with its own URL-based identifier (author, text, owntime) - each URL has its own .json file.

This is the step where data collection concludes.

4. At this point, the contents of each URL is available, which are missing from the aforementioned main_output.json, so they are added to this file one by one.

5. At this point, main_output.json is complete as a whole, and a .csv file is generated from it. The project is capable of being scaled and being run in parallel, up to being run every 5-10 minutes to generate this output.

Explanation regarding why the process is divided into these threads:

The most important part is that the steps build on each other, and in each step the requests/responses and the file writes are done asynchronously, so they need to be executed in this order. (e.g., If we want to set that X URL is in the daily top 1 hour news or not, we need to scrape the top 1 hour news with their URLs first.)

Step 1 and 2 runtime takes about 5-7 seconds and there are no disruptions or constraints from the hirkereso server (or robots.txt).

In Step 3 (as well), every request is performed asynchronously, sent with a 0.62 second* delay independently from everything else (including each request), and because the request delivery time can vary, those (as well) need to be handled with a call back one-by-one, and this is why each of them is saved to separate .json files, so they are completed independently from each other: multiple processes cannot write to the same file at the same time, and so each data is merged at the end instead.

*This 0.62 second delay isn't set in stone, this allows for the project to run without running into bandwidth issues or restrictions with the outlets' servers. (There were issues with smaller values.)

The more legacy servers are used, the shorter the scraping runtime becomes.

In this scenario, there's no need to alter the project code, but separate bash/sh scripts will need to be written which handle the .json files, but this is fairly simple. Alternate reliable solutions are also possible if needed.

With the current 1 server setup, the processing speed is approximately 75-90 pages per minute.

Interrupted processes can be restarted where they were stopped at. In regards to article pages, content that has been already scraped isn't scraped again (request won't be sent).

## Installation
Scrapy installation: pip install should be performed in an activated environment. [how to](https://docs.scrapy.org/en/latest/intro/install.html#ubuntu-14-04-or-above)
```bash
$ git pull (...repo...)
$ python3 -m venv venv
$ pip3 install -r requirements.txt
```
## Usage

The above processes, commands are collected in a run.sh script, this was created with the 1 server setup in mind. When doing test runs, the environment must be always activated beforehand.
```bash
source venv/bin/activate
```
It's enough to run run.sh afterwards:
```bash
$ ./run.sh
```
The script looks like this:
```bash
#!/bin/bash

GREEN='\033[0;32m'
NC='\033[0m'

rm -f ./mediaScraper/temp/*.json &&
  scrapy crawl top_pages -o ./mediaScraper/temp/top_pages_output.json &&
  scrapy crawl main -o ./mediaScraper/temp/main_output.json &&
  scrapy crawl site_page &&
  python mediaScraper/command/resolver/site_data_resolver.py &&
  python mediaScraper/command/converter/main_result_json_to_csv_converter.py &&
  printf "\n${GREEN}The new CSV has been successfully created!${NC}\n\n"
```
There's nothing of note here, but what can be seen here is that each step can be scheduled and executed from the console (possibly on other, additional servers).

Each row is a separate kernel/Python process, the Scrapy requests within each one are fairly big and are executed asynchronously. If the Linux process exists, that's a confirmation of the process finishing (thanks to the scrapy crawl console tool).

This file has a version for cron runs, which is identical to the above script except it does handle the virtual environment so any user can run the cron job without issues.

## Testing a specific outlet scraper
See [testing.md](https://github.com/webpredict-io/scraper/blob/master/mediascraper/testing.md).

## Conclusions
The biggest constraint is the limitations enforced by each outlet website server, so it's possible that content cannot be downloaded even with the 0.62 second delay (it will be visible in the logs).
### Recommendations
- "owntime" values should be converted to a uniform datetime format per outlet
- The handling of redirects is only implemented so far on a limited level for specific outlets, but it isn't implemented on a wider level, nor universally.
- Concurrent processing of configuration files (YAML) in the same location
- Writing integration tests for test pages (this requires test HTML pages)
- Find a solution for implementing different delay values for each outlet Request object
- The creation of unique text cleaning rules in YAML and applying them in the proper locations. (Unnecessary selectors can be excluded in the YAML config; if text cleaning isn't feasible because of the HTML setup, use the substring-filter process and its config file instead.)
- Solve the import problem in the command files, prevent code duplication
- Worth to consider:  https://docs.scrapy.org/en/latest/topics/autothrottle.html
### References
[https://docs.scrapy.org/en/latest/](https://docs.scrapy.org/en/latest/)
[https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/#installing-virtualenv](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/#installing-virtualenv)
