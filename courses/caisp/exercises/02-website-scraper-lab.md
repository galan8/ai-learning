# Exercise: Building a Simple Website Scraper

Course: CAISP (Practical DevSecOps)
Status: Complete (all steps, including production tooling review)

## 1. Objective

Build a small Python scraper using `requests` and BeautifulSoup that fetches a web page, extracts text from the tags that usually hold content, and returns a capped number of characters. The lab notes this tool will be reused in later chapters, so it is a building block rather than a one off.

Why an AI security course teaches scraping: LLMs are trained on large corpora drawn largely from public websites, and attackers use scraping to harvest training data, probe AI systems and extract sensitive information. Understanding how scrapers work is needed to recognise scraping based attacks and to reason about the attack surface of AI systems that depend on web data.

## 2. Environment and tools

1. Lab environment (Linux), Python 3, pip
2. `beautifulsoup4==4.13.3`, `requests==2.32.3`, `urllib3==2.3.0`
3. No model in this lab, so it runs as a plain script with command line arguments rather than a loop

## 3. Steps

### 3.1 Understanding where content lives in HTML

Page text is not confined to one tag. Content typically sits inside `div`, `p`, `span`, headings `h1` through `h6`, and the semantic `article` and `section` tags, nested within `body` and `html`. A general scraper therefore has to collect from all of them.

### 3.2 The scrape function

```python
import requests
import sys
from bs4 import BeautifulSoup

def scrape(url, max_chars):
    response = requests.get(url)
    soup = BeautifulSoup(response.text, 'html.parser')

    text_content = []
    for tag in soup.find_all(['p', 'div', 'span', 'h1', 'h2', 'h3',
                              'h4', 'h5', 'h6', 'article', 'section']):
        if tag.get_text(strip=True):
            text_content.append(tag.get_text(strip=True))

    full_text = '\n'.join(text_content)
    return full_text[:max_chars]
```

`requests.get` fetches the raw HTML, BeautifulSoup parses it into a navigable tree, `find_all` collects the content bearing tags, and `get_text(strip=True)` pulls the visible text out of each one. The results are joined and truncated to `max_chars`.

### 3.3 Main function with input validation

```python
if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Usage: python3 simple_scraper.py <URL> <max_characters>")
        sys.exit(1)

    url = sys.argv[1]
    if not url.startswith("http"):
        print("Error: Invalid URL")
        sys.exit(1)

    try:
        max_chars = int(sys.argv[2])
    except ValueError:
        print("Please enter a valid integer for max_characters.")
        sys.exit(1)

    web_content = scrape(url, max_chars)
    print(web_content)
```

Three checks: correct argument count, a URL that starts with `http`, and a second argument that parses as an integer.

### 3.4 Testing

| Command | Result |
| ------- | ------ |
| no arguments | usage message |
| URL only | usage message (argument count check) |
| `ftp://...` with 100 | Error: Invalid URL |
| valid URL with `xxx` | integer validation error |
| valid URL with 100 | first 100 characters of page text |
| valid URL with 1000 or 10000 | proportionally more content |

Working as intended on simple, server rendered pages.

## 4. Known limitations (from the lab)

1. JavaScript heavy sites will not render, since only the initial HTML is fetched
2. Rate limiting and bot detection can block requests easily
3. Authentication, cookies and session management need substantial extra code
4. Scaling to thousands of pages becomes complex and resource intensive
5. It only works where content sits in the specific tags being targeted

## 5. Production alternatives (from the lab)

1. **Firecrawl**: renders pages in real browsers, so it captures JavaScript loaded content, converts pages to clean markdown or JSON, can click, scroll and submit forms, and parses PDF and DOCX. Suited to single page applications and preparing web content for AI pipelines.
2. **Apify**: cloud platform built around reusable programs called Actors, with proxy management, scheduling, storage and webhooks, plus a marketplace of pre built scrapers. Suited to large scale, ongoing collection.
3. **Exa.ai**: neural search that indexes pages by semantic meaning rather than keywords, returning links, full text, highlights or summaries through an API. Often used as a retrieval backend for RAG.

Choosing between them: Firecrawl for JavaScript heavy sites needing clean structured output, Apify for enterprise scale pipelines and pre built integrations, Exa.ai when semantic relevance matters more than exact page extraction.

## 6. Lessons learned (security focus)

1. **The input validation here is a good illustration of validation that looks sufficient but is not.** `url.startswith("http")` also accepts `httpfoo`, and more importantly it permits any host, so `http://127.0.0.1:8080/admin` and `http://169.254.169.254/latest/meta-data/` both pass. This is SSRF, the same weakness as in the summarizer lab, and it is worth noting that a check exists here and still does not close it. Proper validation parses the URL, restricts the scheme to http and https explicitly, resolves the host, and rejects private, loopback, link local and metadata ranges.
2. **No timeout on `requests.get` means the process can hang indefinitely** if a server never responds. `requests` has no default timeout. Always pass one.
3. **No size limit on the response.** `max_chars` truncates the output after the whole page has already been downloaded and parsed into memory, so it is a display limit, not a resource control. A large or deliberately oversized response can exhaust memory.
4. **This is the data collection step of the ATLAS chain.** The Reconnaissance and Collection tactics from the Chapter 2 notes are exactly this activity performed at scale: harvesting public material about a target's AI systems. The same tool serves defenders monitoring their own exposure and attackers mapping it.
5. **Scraped content is untrusted input, and here it becomes model input.** The lab says this scraper will be reused later in the course, which means its output will feed a model. That is the indirect prompt injection path again: text on a page becomes text in a prompt, with no boundary marking it as data rather than instructions. Anything reused as a component should carry that warning with it.
6. **Legal and ethical dimension the lab does not raise.** Scraping is governed by terms of service, robots.txt, rate limits and, for personal data, data protection law. A scraper that ignores robots.txt and sends unthrottled requests can constitute abuse regardless of intent. Worth encoding as habit: respect robots.txt, identify your agent, throttle, and have a lawful basis before collecting personal data.
7. **Attacker use of scraping runs both directions.** Attackers scrape to build training data and profile targets; they also poison the pages that scrapers collect, knowing the content may end up in a training corpus or a RAG index. Being on the collecting end does not make you safe.

## 7. Ideas to take forward

1. Experiment idea: harden this scraper. Parse the URL properly, restrict schemes, block private and link local address ranges after DNS resolution, add a timeout, cap the download size with streaming, set a descriptive user agent, and honour robots.txt
2. Experiment idea: compare output against a JavaScript rendered page to see the limitation concretely
3. Concept file candidate: `concepts/ssrf-in-ai-tooling.md`, using the summarizer and this scraper as the two worked examples
