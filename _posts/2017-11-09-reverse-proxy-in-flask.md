---
layout: post
title: Reverse Proxy in Flask
---

Sometimes one wants to present external pages as part of one’s own website. 
For example, I have a wiki on a third-party site that I want to navigate to 
when a user clicks on the “Wiki” link on my website. The catch is that 
I don’t want the URL shown in the browser to show `bitbucket.org`. 

It’s really easy to accomplish this in Flask. As a bonus, you can use 
BeautifulSoup to edit the page on-the-fly before returning it to the client. 
In my case, I removed sidebars and unneeded hyperlinks from the page.
This also works for proxying HTTPs sites.

The code below is a simplified version of `FlaskProxy` on GitHub.
It does not handle some corner cases, nor does it set the headers,
such as `X-Forwarded-For` properly. I recommend you check out
[FlaskProxy](https://github.com/tushar-mohan/flaskproxy), 
but you might want to skim the code below for a high-level understanding.

Here is the relevant Python function with the Flask route decorator:

```
@app.route('/wiki/)
def wikiproxy(p = ''):
    import requests
    url = 'https://ENTER-EXTERNAL-WEBSITE-URL/{0}'.format(p)
    try:
        r = requests.get(url)
    except Exception as e:
        return "proxy service error: " + str(e), 503

    # You can edit the page to your heart's content
    # or just return r.content without parsing
    from bs4 import BeautifulSoup
    soup = BeautifulSoup(r.content, "html.parser")

    # remove sidebar
    soup.find('div', id="adg3-navigation").decompose()
    # remove all buttons with specified class
    selects = soup.findAll('a', class_="aui-button")
    for match in selects:
        match.decompose()
    # remove header breadcrumbs
    soup.find('header', class_="app-header").decompose()
    soup.find('title').string = 'My Wiki'

return str(soup)
```
