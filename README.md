# Awarri1
webscraping

### 🧭 **High-Level Purpose**

This R script:

1. Opens a Chrome browser using RSelenium.
2. Visits [https://www.ibdb.com/shows](https://www.ibdb.com/shows).
3. Scrapes show data from three tabs (Current, Upcoming, Past).
4. Clicks and bypasses a CAPTCHA checkbox.
5. Extracts show info: title, link, image, theatre name/venue, date, performance, and type.
6. Saves data to a CSV file named by today’s date.
7. Checks for duplicate data based on title + theatre name.
8. If new entries are found, sends a Slack message using the web app.

---

### 🧱 **1. Setup and Dependencies**

```r
library(RSelenium)
library(rvest)
library(stringr)
library(tidyverse)
library(magrittr)
library(pdftools)
library(ralger)
```

These libraries are loaded to handle:

* `RSelenium`: controls the browser.
* `rvest`, `stringr`, `tidyverse`, `magrittr`: web scraping and data wrangling.
* `pdftools`, `ralger`: unused in this snippet but likely included for file parsing and general web scraping utilities.

---

### 🚗 **2. Launch Browser with RSelenium**

```r
remDr <- rsDriver(browser = "chrome", chromever = "119.0.6045.105", verbose = FALSE)$client
```

Starts a Chrome browser using a matching ChromeDriver version (`119.0.6045.105`). `$client` gives you access to browser control commands.

---

### 🌐 **3. Navigate to IBDB**

```r
remDr$navigate("https://www.ibdb.com/shows")
```

Navigates the browser to the Broadway show listings.

---

### 🧭 **4. Extract All Tabs: Current, Upcoming, Past**

```r
tabs <- remDr$findElements(using = "xpath", value = "//app-show-index-tabs//a")
```

Finds all tab elements (like "Current Shows", "Upcoming", "Past") using XPath. This returns a list of three clickable tab objects.

---

### 🧼 **5. Click and Bypass CAPTCHA Checkbox**

```r
checkbox <- remDr$findElement(using = "css", value = "label[for='recaptcha-anchor']")
checkbox$clickElement()
```

Attempts to bypass the CAPTCHA by clicking the checkbox. **Note:** This works only for very simple reCAPTCHAs. Most reCAPTCHAs will still block bots.

---

### 🕸️ **6. Loop Through Each Tab to Collect Data**

```r
results <- list()
```

Initialize an empty list to hold data from each tab.

---

#### 🔁 **For each tab (current, upcoming, past):**

```r
for(i in 1:length(tabs)){
    remDr$navigate("https://www.ibdb.com/shows")
    Sys.sleep(2)
```

Navigates back to the page and waits 2 seconds (helps reduce errors due to loading delays).

```r
    tabs <- remDr$findElements(using = "xpath", value = "//app-show-index-tabs//a")
    tabs[[i]]$clickElement()
    Sys.sleep(2)
```

Re-finds the tab elements (fresh copy) and clicks the `i-th` tab. Waits again for it to load.

```r
    showcards <- remDr$findElements(using = "css", value = "app-show-card.ng-star-inserted")
```

Finds all show cards currently visible under the selected tab.

---

### 📥 **7. Extract Show Data for Each Card**

```r
titles <- c()
images <- c()
links <- c()
theatres <- c()
theatre_venue <- c()
dates <- c()
performance <- c()
type <- c()
```

Initializes empty vectors to store show info for each tab.

---

#### 🔁 **Loop through each show card:**

```r
for(show in showcards){
    html <- show$getElementAttribute("outerHTML")[[1]]
    html <- read_html(html)
```

Extracts the HTML from each show card and parses it.

```r
    titles <- c(titles, html %>% html_element("h3") %>% html_text(trim = TRUE))
    links <- c(links, paste0("https://www.ibdb.com", html %>% html_element("a") %>% html_attr("href")))
    images <- c(images, html %>% html_element("img") %>% html_attr("src"))
```

Scrapes:

* Title (`<h3>`)
* Show description page link
* Image URL

```r
    details <- html %>% html_elements("ul li span") %>% html_text(trim = TRUE)
    theatres <- c(theatres, details[1])
    theatre_venue <- c(theatre_venue, details[2])
    dates <- c(dates, details[3])
    performance <- c(performance, details[4])
    type <- c(type, details[5])
}
```

Extracts five other text fields from the `<ul><li><span>` tags:

* Theatre name
* Venue
* Date
* Performance status
* Show type (e.g. Musical, Play)

---

### 📦 **8. Combine Tab Data into a DataFrame**

```r
results[[i]] <- data.frame(
    title = titles,
    Description_link = links,
    image_urls = images,
    theatre_Name = theatres,
    theatre_venue = theatre_venue,
    Date = dates,
    performance = performance,
    type = type,
    scrape_time = Sys.time()
)
}
```

Each `results[[i]]` contains a dataframe for one tab. The `scrape_time` records when the data was scraped.

---

### 🧺 **9. Merge and Save All Data**

```r
result <- bind_rows(results)
```

Combines all three dataframes into one.

```r
write.csv(result, paste0("ibdb_shows_", Sys.Date(), ".csv"))
```

Saves it with today’s date in the filename.

---

### 🧠 **10. Deduplication Check**

```r
if(file.exists("shows.csv")){
    old <- read.csv("shows.csv")
    new <- anti_join(result, old, by = c("title", "theatre_Name"))
    write.csv(result, "shows.csv")
} else {
    write.csv(result, "shows.csv")
}
```

Checks if `shows.csv` already exists:

* If **yes**, compares the new data to the old.
* Saves only new entries (compared by `title` + `theatre_Name`).
* If **no**, creates the file.

---

### 💬 **11. Slack Notification if New Data Found**

```r
if(nrow(new) > 0){
    remDr$navigate("https://slack.com/app_redirect?channel=...")  # your Slack channel
    Sys.sleep(10)
```

If there’s new data, go to a Slack channel (via browser Slack Web).

```r
    message <- remDr$findElement(using = "xpath", value = "//*[@id='msg_input']/div[1]/div/div[2]/div")
    message$sendKeysToElement(list(paste0(nrow(new), " new Broadway shows added today! 🎭")))
    message$sendKeysToElement(list(key = "enter"))
}
```

Types and sends a message into the Slack input box:
`"X new Broadway shows added today! 🎭"`

---

### 🛑 **12. End the Browser Session**

```r
remDr$close()
```

Closes the RSelenium browser.

---

### ✅ **Wrap-up**

This script:

* Uses RSelenium to interact with a real browser
* Clicks and navigates dynamic tabs
* Bypasses simple CAPTCHA
* Scrapes full Broadway show listings from IBDB
* Checks for new additions and alerts via Slack
* Outputs structured CSV logs per day

Let me know if you want a version that runs headlessly, uploads to a database, or uses Slack Webhooks for cleaner integration.


checking the robots.txt file

#the robots.txt file

#user-agent: *
 # disallow: /replacements/
#  disallow: /grosses-production/
 # disallow: /show-audio-interview/
 # disallow: /cast-staff-audio-interview/
 # disallow: /tour-production-more/
 # disallow: /broadway-production-more/
 # sitemap: https://www.ibdb.com/sitemap.xml



 NB; for periodical scheduling we wont use code but schedule through the windows task scheduler it self how to do it is to save the file,
 open the scheduler, schedule the program and run 
 
