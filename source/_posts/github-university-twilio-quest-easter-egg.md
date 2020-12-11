---
title: Operation Univers(ity) TwilioQuest Easter Egg
tags:
  - GitHub
  - Twilio
categories:
  - U6C61
date: 2020-12-10 16:42:48
abstract: "There's an easter egg within <a href=\"https://www.twilio.com/quest\" target=\"_blank\">TwilioQuest</a>. Your first hint is: \"hit the nail on the head.\""
message: "<strong>SpOiLeR aLeRt</strong>: before I reveal how I found the easter egg, I strongly recommend you to try this yourself to see if you can \"<a href=\"https://github.com/education/github-university-2020\" target=\"_blank\">hit the nail on the head</a>.\" When you are ready, use the password \"<strong><em>Adventure</em></strong>\" (without the quotes) below to unlock:"
password: Adventure
---

{% note warning %}
I wanted the adorable [Octoplush](https://github.myshopify.com/products/octoplush) for so long, and [it's finally on the way](https://www.twitch.tv/videos/832648555)!

![Octoplush is a cuddly Octocat](https://cdn.shopify.com/s/files/1/0051/4802/products/Octoplush_d76ba290-d65b-40b9-a675-cfc3afa76b6e_large.JPG)
{% endnote %}

## A "Slight" Detour

Since the hint is "hit the nail on the head," I believe a lot of people have tried to "hit" everything that has even just the slightest connection to either "nail" or "head".

Me too. I spent countless hours trying to click the head of the 2 people in the welcome screen, that of robot Cederic, the head of the Operator, and many other places. Yet, none of those worked.

Well, time to try reverse engineering. Let's see if the source code of the app can tell us something.

{% note danger %}
Note: make sure you are allowed to do this by the Terms of Service / End User License Agreement.
{% endnote %}

You can view the source code of TwilioQuest by selecting the Help menu, then Toggle Developer Tools.

![TwilioQuest with its source code after toggling developer tools](https://user-images.githubusercontent.com/10842684/101840154-9712b300-3b11-11eb-87e7-b3001729e8d1.png)

For easter eggs/hidden features, the words that I usually start to look for in source code are "easter," "egg," "secret," "found," and "congratulations." Since the hint for this one is "hit the nail on the head", I also tried to search for "nail," "head," "hit," and "click" in all of TwilioQuest's source files. (Somehow I can never remember how to search for all files using Chrome Developer Tools, so here's a [stack overflow post](https://stackoverflow.com/questions/37685351/chrome-devtools-search-all-javascript-files-in-website/47690078) demonstrating how.) Yet, nothing of particular interest showed up in the results.

## "Nail" in the `head`

When I checked the [modification history of the README.md file](https://github.com/education/github-university-2020/commits/main/README.md) (to see if they have accidentally leaked something), a version of it said that "the game will let you know if you've qualified!" So I assumed that this must be a feature in the TwilioQuest app itself, revealed by some kind of JavaScript code / hidden chests / etc....

But what if it is not? Since Ekansh Gupta mentioned about opening TwilioQuest source code on Slack, we're probably in the right place. But we've spent too much time looking for the clue, how could Vicdoja solve the puzzle in less than an hour after receiving the hint on Slack? It must be something that's directly related to the hint, something obvious that we've been ignoring.

Well, indeed there was. There's something that we've ignored so far: the `head` HTML tag seen in the first screenshot. Because there's no "code" in the `head` tag, I never considered it to be the head that I was looking for. With no where else to examine, I expanded the `head` tag by clicking that triangle to its left:

![There's a comment saying follow the white rabbit with a link to GitHub gists](https://user-images.githubusercontent.com/10842684/101842641-7a2cae80-3b16-11eb-823d-b4251724b0d8.png)

We finally hit the nail on the head and can "follow the white rabbit" to continue our adventure.

## More Puzzles

<script src="https://gist.github.com/kwhinnery/b3cbc250fa1df65cc3c36ca39495d486.js"></script>

The file is named "[egg.md](https://gist.github.com/kwhinnery/b3cbc250fa1df65cc3c36ca39495d486)" and the description says "GitHub Univers(ity)," indeed this is the easter egg that we've been looking for. But what could this long line of cryptic text possibly mean?

If you scroll to the end of the line, you'll see "`==`". That's a classic sign of Base64 encoded text, so I just searched for an online Base64 decoder and decoded this weird line of characters, yielding:

{% note info %}
To complete the egg hunt for GitHub Univers(ity) 2020, email kwhinnery@twilio.com with a subject line containing [EGG HUNT], with the answer to this question:

What game, written by Warren Robinett, is credited as having the first-ever easter egg?
{% endnote %}

Thanks to the Internet and technologies we have today, this should be a simple question to answer.

## cont.

Although [GitHub Universe](https://githubuniverse.com/) 2020 is over, it's not too late to start contributing to the open source community. Tryout "[The Flame of Open Source](https://www.twilio.com/quest/learn/open-source)" training mission in TwilioQuest to help you get familiar with the process, and adventure on! Together, we can save the flame of open source 🔥.

![Operation Univers(ity), an epic journey to save the flame of open source](https://user-images.githubusercontent.com/6633808/101407837-c50eb200-38db-11eb-90ff-4888b5598de0.png)
