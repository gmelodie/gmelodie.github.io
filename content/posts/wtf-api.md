+++
public = "true"
date = "2024-08-07"
title = "WTF is an API? (or, \"Everything's a Lie\")"
+++

This is a rewrite of an older post of mine ([this one](https://gmelodie.medium.com/get-started-in-web-development-with-me-protocols-servers-and-http-33c527725b74)). It bothers me that a lot of people explaining APIs throw around loads of confusing, meaningless buzzwords, to an otherwise easily explainable topic. It feels like these people are just trying to sound smart instead of explaining APIs. The reason why this post could also be called "Everything's a Lie" is because a lot of what we'll be doing is proving that most things you see around about APIs are just lies.

In this post, we'll explore the meaning and reasoning behind the idea of an API, see some examples, and talk about REST. Oh, and we'll be doing some coding as well, as always.


Shall we?


## The naming
First things first, let's talk about the acronym. API stands for **Application Programming Interface**, what does that mean?

The important part is **Interface**. An interface is anything that sits between two other things. We call the USB port a USB *interface* because it lets the computer communicate with peripherals (or vice-versa). Interfaces are important because they enforce a **protocol** that both ends must speak. It doesn't matter how the mouse or the computer are built, if they communicate through USB, they should follow the USB standard. **Application Programming** means creating an interface for applications to talk to each other, making *application programming* easier.

> **Protocols** in the computer world are a set of rules that devices should adhere to to communicate. Something like: "To adhere to the USB standard, you must send three bytes with values `x`, `y` and `z`, then you should wait for the other device to send value `a1`, etc."
>
> **e.g.:** In a valid HTTP request message, the first few characters must be the method (`GET`, `POST`, `DELETE`, etc) followed by a whitespace, followed by a path (`/users`, `/`, etc) followed by... You get the idea.


If you have ever read anything about APIs you might be wondering: what does this have to do with HTTP or GET or POST? And the answer is: nothing! Really, nothing. In theory, there's nothing that dictates that APIs have to be implemented using HTTP.

If you've never read anything about APIs before, however, the last paragraph made no sense whatsoever. Let's rewind a bit.

## SDKs are APIs

To explain what I mean by `SDKs are APIs`, let's implement a simple Rust library to print ASCII characters enclosing text, like so:

```rust
let my_text = "hello world";
enclose(my_text, '#');
```

The output for the above should be:
```
###############
# hello world #
###############
```

Here's the full Rust source code for the curious:
```rust
fn enclose(text: &str, enclose_char: char) {
    println!("{}", enclose_char.to_string().repeat(text.len() + 4));

    println!("{enclose_char} {text} {enclose_char}");

    println!("{}", enclose_char.to_string().repeat(text.len() + 4));
}

fn main() {
    let my_text = "hello world";
    enclose(my_text, '#');
}
```

Now let's say that we want other people to use our code, for that we'd make this into a lib and the code wouldn't look much different:
```rust
pub fn enclose(text: &str, enclose_char: char) {
    println!("{}", enclose_char.to_string().repeat(text.len() + 4));

    println!("{enclose_char} {text} {enclose_char}");

    println!("{}", enclose_char.to_string().repeat(text.len() + 4));
}
```

We just deleted the main function and added `pub` to make the `enclose()` function public. You'd need to alter some things in the package configs as well, but the point is that if we *actually* want our code to be modular and usable, there are much more important changes we need to do:
```rust
pub fn enclose(text: &str, enclose_char: char) -> String {
    let horizontal_line = enclose_char.to_string().repeat(text.len() + 4);
    format!("{horizontal_line}\n{enclose_char} {text} {enclose_char}\n{horizontal_line}")
}
```

The important change is that we return the new enclosed string instead of printing it to the standard output. This is important because if we want people *programming applications* we need to make our code easily pluggable. This new version allows programmers to get the strings and do whatever they want with it. Maybe they don't want to print the string, maybe they want to plot it as a 3D model instead!

What we wrote is a library (lib, for short), but it could also be called an SDK: a **Software Development Kit**. SDKs are just a collection of functions that allow you to develop something easier. We usually don't call every lib an SDK, but they're the same thing. Purists will say they're not. They're wrong. Purists are bikeshedding dicks. Do not trust purists.

Per our previous definitions, an SDK is therefore an API: it's code on top of which people build applications. Now, let's explore what people *usually* mean when they talk about APIs.


## What people usually mean when they talk about APIs

In the early days of the internet websites were dead stupid: static HTML, manually updated from time to time.

> **Static HTML** pages are what we call HTML that's in a file in the server (the HTTP server, also called a "web server"), which hands clients a copy of that file. **Dynamic** pages on the other hand contain code and style sheets, usually JavaScript and CSS respectively, that allow the client to *dynamically* generate the "final" HTML. With dynamic pages, the client downloads HTML templates, JS, CSS, and then generates one big HTML by crunching those three together.

Let's say we wanted to write a bot to get the news titles from your local newspaper website back in 2001. Back then, you'd probably have to download the HTML for the entire page, parse it, and spit out only the titles[^1]. Here's what that would look like in Python (in this case we're parsing [motherfuckingwebsite.com](http://motherfuckingwebsite.com)):

[^1]: [RSS](https://wikipedia.org/wiki/RSS) wasn't very popular then.

```python
import requests
import re

text = requests.get("http://motherfuckingwebsite.com/").content.decode('utf-8')

pattern = re.compile(r'<h2>(.*?)</h2>', re.DOTALL)

# Find all text between <h2> and </h2> in the HTML content
titles = pattern.findall(str(text))

for t in titles:
    print(t)
```


Output:
```
Seriously, what the fuck else do you want?
It's fucking lightweight
It's responsive
It fucking works
This is a website. Look at it.  You've never seen one before.
```

This approach is (1) wasteful because we have to download many characters (the entire page!) only to get a few titles and (2) the code is hard to maintain because we have to figure out where the titles are in the HTML (in this case they're on the `h2` tags), and if they decide to change the layout of the website, we'd have to rewrite our bot.

So when people talk about APIs, they usually mean **HTTP APIs**, which is a way to facilitate getting information for your website. Let's write an API for [motherfuckingwebsite.com](http://motherfuckingwebsite.com):

```python
from flask import Flask, jsonify
import requests
import re

app = Flask(__name__)

@app.route('/titles', methods=['GET'])
def get_titles():
    text = requests.get("http://motherfuckingwebsite.com/").content.decode('utf-8')

    pattern = re.compile(r'<h2>(.*?)</h2>', re.DOTALL)

    # Find all text between <h2> and </h2> in the HTML content
    titles = pattern.findall(str(text))

    # Return the titles as JSON
    return jsonify(titles)

if __name__ == '__main__':
    app.run(debug=True)
```

Here's what we do:
1. Get the raw HTML
2. Spawn a webserver of our own
3. Make the list of titles available in JSON by `GET`ting `/titles`

If our website were called `amazingwebsite.com` then you'd only have to `GET http://amazingwebsite.com/titles` to get the JSON with the titles.


## REST: Representational State Transfer

Over time people started getting clever and realized that they could make their APIs *receive* data from the applications/users instead of only giving it away. A canonical example is Twitter's API, which allows programmers to create bots that send data (like a bio update or a tweet) to the server.

Here's what a bio update looks like:
```bash
curl -X POST https://api.twitter.com/1.1/account/update_profile.json?name=Sean%20Cook&description=Keep%20calm%20and%20rock%20on
```

HTTP uses **verbs/methods** to allow us to do different things with a server. If we're requesting information, that's typically a `GET` request, but in the example above we're actually sending data to the server, so we use the `POST` method. The cool thing is that we can have different behaviors for the same route by using different methods:

```
# get data on the gmelodie account
curl -X GET mywebsite.com/accounts/gmelodie

# delete the gmelodie account
curl -X DELETE mywebsite.com/accounts/gmelodie

# update the description of the gmelodie account to newDescription
curl -X POST mywebsite.com/accounts/gmelodie?description=newDescription
```

That's basically what a REST API is. Again, purists will say there's much more to REST APIs, but the gist of it is: REST APIs are APIs that do different things based on the HTTP method you pass to a route.


## Appendix I: Idempotency
From [dictionary.cambridge.org](https://dictionary.cambridge.org/dictionary/english/idempotent):
> An idempotent element of a set does not change in value when multiplied by itself.

In programming, a function is idempotent if, when called twice with the same parameters, the operation only happens once. This is especially interesting when building APIs. For instance, if a client makes two requests to send the same tweet, they probably don't want to post the same tweet again.

Or do they? Maybe they do! To make sure the client wants to do the same thing twice, you can add an [`idempotency-key`](https://docs.bankly.com.br/docs/idempotency-key) field to the requests. If the server gets two identical requests to post the same tweet with the same `idempotency-key`, it'll disregard one of them. Now, if the server receives two identical requests to post a tweet, but with two different `idempotency-key`s, it'll post both tweets.

---
Thank you for reading!

## References
- [Twitter: POST account/update_profile](https://developer.x.com/en/docs/twitter-api/v1/accounts-and-users/manage-account-settings/api-reference/post-account-update_profile)
- [Chave de idempotência](https://docs.bankly.com.br/docs/idempotency-key)
- [What Is an Idempotent API?](https://blog.hubspot.com/website/idempotent-api)

