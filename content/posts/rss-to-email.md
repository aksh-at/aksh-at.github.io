---
title: "\"Substack for RSS feeds\", built using Clojure and Datomic Ions"
date: 2021-12-27T20:20:25-07:00
draft: false
type: post
---

[Jessica](https://twitter.com/jasshikoo) and I spent holiday weekend building https://rssto.email, a feed subscription service written in Clojure and hosted entirely on [Datomic Ions](https://docs.datomic.com/cloud/ions/ions.html). 

The project started out as an Ion I wrote a few weeks ago, because _none_ of the existing ways to subscribe to RSS feeds had the user experience I really wanted. The result was useful enough that we decided to productionize it and share the source ([repo](https://github.com/aksh-at/rss-to-email)).

There's not a lot of user stories about Ions out there (despite [intermittent HN hype](https://news.ycombinator.com/item?id=17254222)), so I'm also sharing some of my takeaways here from building on this stack.

![rssto.email](/img/rsstoemail.png)

# Why build this?

There's a lot of incredible blogs and writers on the internet, but at least for me, it's far easier to keep up with the ones that have email subscriptions, either through Substack or without. Most blogs have RSS/Atom feeds but none of the existing RSS readers I've tried have given me a functional, ad-free, made-for-email-reading experience that I really want. This project is mostly a way to solve that problem for myself.

In general, I'm pretty excited by how the number of interesting voices and ideas on the internet is growing so rapidly, and I think there's a lot more that you could do with this:
- share your subscriptions
- better discovery
- recommendations
- a "catch me up" mode that drip feeds you the back catalog

I don't have time at present to build those things, but if you're interested in contributing, I'd love to talk to you!

# Why Clojure & Datomic?
Quick background: [Clojure](https://clojure.org/) is a Lisp-like functional programming language that runs on the JVM; [Datomic](https://www.datomic.com/) is a distributed database that's queryable using [Datalog](http://www.learndatalogtoday.org/) and has "[time-travel](https://docs.datomic.com/cloud/tutorial/history.html)" built in; [Datomic Ions](https://docs.datomic.com/cloud/ions/ions-tutorial.html) is a Datomic offering that lets you deploy functions serverlessly as AWS Lambdas.

I've been interested in trying out Ions since I first heard about them—REPL-driven development is great, and the ability to write code and deploy it to production from my REPL without worrying about infra would be game-changing. 

Fun tech aside, a serverless back-end that uses a horizontally scalable database seemed like the perfect architecture for a side project—easy to keep running even if I'm the only user, and trivial to scale up if not.

# Takeaways

### Emacs magic

The emacs ecosystem has a ton of support for working with Lisps (not surprising since [emacs itself is written in a Lisp](https://en.wikipedia.org/wiki/Emacs_Lisp)), and investing a small amount of time in setting up tooling actually goes a long way:

- [Evil-cleverparens](https://github.com/luxbock/evil-cleverparens) makes parentheses/sexp manipulation a breeze. I'm not going to go into all the shortcuts here, but here's a demo (regular vim `dd` deletes a couple lines now while _keeping parentheses balanced_ and then `>` moves the parentheses to enclose the adjacent expression):
![evil-cleverparens action](/img/clj-cleverparens.gif)
- [Clojure LSP](https://github.com/clojure-lsp/clojure-lsp) lets you do fancy AST manipulation automatically, such as converting nested functions to use [threading macros](https://clojure.org/guides/threading_macros) (which are one of the things about the language that make it so ergonomic):
![Clojure LSP in action](/img/clj-thread.gif)

- Out-of-the-box linting doesn't do that much, and _aggressive mode_ feels a lot better:
{{< highlight clj >}}
(add-hook 'clojure-mode-hook #'aggressive-indent-mode)
(setq clojure-indent-style 'align-arguments)
(setq clojure-align-forms-automatically t)
{{< / highlight >}}

### Clojure is powerful
Since Lisps [have no syntax](http://www.paulgraham.com/avg.html), you can create syntactic constructs on the fly that help you best express whatever you're writing. While not a crazy example, a cute illustration of this is in [hiccup](https://github.com/weavejester/hiccup), the library used to create html templates for emails. It lets you manipulate expressions that get compiled to html, and that ends up feeling basically like writing React:

{{< highlight clj >}}
(defn format-notify-body [feed-url new-posts manage-link]
  (html [:html
         [:body
          [:div (seq  (mapv post-html new-posts))]
          [:div
           [:a {:href manage-link :style {:color "blue"}}
            "Unsubscribe"]]]]))
{{< / highlight >}}

I had hoped to get my hands dirty with the language's concurrency model, but parallelizing the core polling loop was as easy as replacing `map` with `pmap`, so I guess that's nice!

Given that it's a functional language, I was surprised at the amount of support for mutating references, especially useful for concurrent access to shared memory—e.g. you can build memoization using [atoms](https://clojure.org/reference/atoms) and transactional updates using [refs](https://clojure.org/reference/refs). Atoms made it pretty easy to to build a mock function that records the args it was called with, that probably would have been quite difficult in a more "pure" functional language.

Pretty much the only complaint I have with the language is that JVM start-up time really hurts. It takes ~10s to do anything (run tests, start up a REPL, deploy to Datomic). I hacked together a [deploy script](https://github.com/aksh-at/rss-to-email/blob/master/deploy.clj) just so I can push and deploy without having to be in the loop. (Side note: the script uses [babashka](https://github.com/babashka/babashka), an interpreted Clojure runtime for scripting, which looks like an awesome project.)

### Datomic Ions are getting there

As for Datomic Ions themselves, they do the stated job: deploying a function to an AWS Lambda is actually just 2 CLI commands. I was especially impressed that I spent no time on package or dependency bullshit, which is the norm if in Python land (which is where I've been recently).

However, it turns out when you actually want to build a production service, you need a lot more. I had to spend a bunch of time figuring out:
- Where to find the logs from my running service
- How to print my own logs (`println`'s don't get persisted)
- How to proxy my lambdas in AWS API gateway (the docs bait you into thinking you can use Ion's HTTP Direct feature, but it turns out you can only deploy _one_ ion that way, and then using that one function to manually dispatch a bunch of hundlers is just gross)
- The type signatures of some of Datomic's API—some of them are not documented anywhere, and Datomic is not open source
- How to back up my DB (turns out [you can't yet](https://forum.datomic.com/t/cloud-backups-recovery/370))

Overall, it seems like the product is at the stage where it makes for a very compelling demo, but there's a bit more polish needed to make it truly magical, especially for first-time users. That said, I'm pretty excited by where things are headed, and that more languages and ecosystems are trying to build serverless abstractions.

The good news is that since I've spent the fixed cost of figuring out some of this extra boilerplate, you can pretty much copy it wholesale from the [github repo](https://github.com/aksh-at/rss-to-email) for your own project. For the [same reason](https://en.wikipedia.org/wiki/Sunk_cost), I think there's a good chance I'll be back to build more things on this stack.
