# Table of Contents

In this tutorial you will learn how to build a Bot with R and `telegram.ext`, with the following sections:

- [Introducing `telegram.ext`](#introducing-telegramext)
- [Building a Bot in 3 steps](#building-a-bot-in-3-steps)
- [Adding Functionalities](#adding-functionalities)

To begin, though, you'll need to create a Telegram Bot in order to get an Access Token. You can do so by talking to [@BotFather](https://telegram.me/botfather) and following a few simple steps (described [here](https://core.telegram.org/bots#6-botfather)).

# Introducing `telegram.ext`

The `telegram.ext` package is built on top of the pure API implementation. It provides an easy-to-use interface and takes some work off the programmer. It uses `telegram` package methods to connect to the API and is based on the `python-telegram-bot` library, using nomenclature from its `telegram.ext` submodule. 

It consists of several `R6` classes, but the two most important ones are `Updater` and `Dispatcher`.

The `Updater` class continuously fetches new updates from Telegram and passes them on to the `Dispatcher` class. If you create an `Updater` object, it will create a `Dispatcher`. You can then register handlers of different types in the `Dispatcher`, which will sort the updates fetched by the `Updater` according to the handlers you registered, and deliver them to a callback function that you defined. Every handler is an instance of any subclass of the `Handler` class.

# Building a Bot in 3 steps

With that said, let's *get started*!

## 1. Creating the `Updater` object

First, you first must create an `Update` object. Replace `TOKEN` with your Telegram Bot's API Access Token, which looks something like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`.

```r
library(telegram.ext)
updater <- Updater(token='TOKEN')
```

**Recommendation:** Following [Hadley's API
guidelines](http://github.com/hadley/httr/blob/master/vignettes/api-packages.Rmd#appendix-api-key-best-practices)
it's unsafe to type the `TOKEN` just in the R script. It's better to use
enviroment variables set in `.Renviron` file.

So let's say you have named your bot `RBot` (it's the first question
you answered to the *BotFather* when creating it); you can open the `.Renviron` file with the R commands:

```r
user_renviron <- path.expand(file.path("~", ".Renviron"))
file.edit(user_renviron) # Open with another text editor if this fails
```

And put the following line with
your `TOKEN` in your `.Renviron`:

```bash
R_TELEGRAM_BOT_RBot=TOKEN
```
If you follow the suggested `R_TELEGRAM_BOT_` prefix convention you'll be able
to use the `bot_token` function (otherwise you'll have to get
these variable from `Sys.getenv`).

After you've finished these steps **restart R** in order to have
working environment variables. You can then create the `Updater` object as:

```r
updater <- Updater(token = bot_token('RBot'))
```

**Recommendation 2:** For quicker access to the `Dispatcher` used by your `Updater`, you can introduce it locally:

```r
dispatcher <- updater$dispatcher
```

## 2. The first function

Now, you can define a function that should process a specific type of update:

```r
start <- function(bot, update){
	bot$sendMessage(chat_id = update$message$chat_id, text = "Hello Creator!")
}
```

The goal is to have this function called every time the Bot receives a Telegram message that contains the `/start` command.
To accomplish that, you can use a `CommandHandler` (one of the provided `Handler` subclasses) and register it in the dispatcher:

```r
start_handler <- CommandHandler('start', start)
dispatcher$add_handler(start_handler)
```

## 3. Starting the Bot

And that's all you need. To start the bot, run:

```r
updater$start_polling()
```

Give it a try! Start a chat with your bot and issue the `/start` command - if all went right, it will reply.

# Adding Functionalities

We have already built a Telegram Bot with R. However, it can now only answer to the `/start` command, so now we are going to add a couple of functionalities, including:

- [Text responses](#text-responses)
- [Commands with arguments](#commands-with-arguments)
- [Unknown command handling](#unknown-command-handling)
- [Stopping the Bot](#stopping-the-bot)

## Text responses

Let's add another handler that listens for regular messages.
Use the `MessageHandler`, another `Handler` subclass, to echo to all text messages:

```r
echo <- function(bot, update){
	bot$sendMessage(chat_id = update$message$chat_id, text = update$message$text)
}

echo_handler <- MessageHandler(echo, Filters$text)
dispatcher$add_handler(echo_handler)
```

From now on, your bot should echo all non-command messages it receives.

**Note:** As soon as you add new handlers to `dispatcher`, they are in effect.

**Note:** The `Filters` object contains a number of functions that filter incoming messages for text, images, status updates and more.
Any message that returns `TRUE` for at least one of the filters passed to `MessageHandler` will be accepted.
You can also write your own filters if you want.

## Commands with arguments

Let's add some actual functionality to your bot. We want to implement a `/caps` command that will take some text as an argument and reply to it in CAPS.
To make things easy, you can receive the arguments (as a `vector`, split on spaces) that were passed to a command in the callback function:

```r
caps <- function(bot, update, args){
	text_caps <- toupper(paste(args, collapse = ' '))
	bot$sendMessage(chat_id = update$message$chat_id, text = text_caps)
}

caps_handler <- CommandHandler('caps', caps, pass_args = TRUE)
dispatcher$add_handler(caps_handler)
```

**Note:** Take a look at the `pass_args = TRUE` in the `CommandHandler` initiation.
This is required to let the handler know that you want it to pass the list of command arguments to the callback.
All handler classes have keyword arguments like this. Some are the same among all handlers, some are specific to the handler class.
If you use a new type of handler for the first time, look it up in the docs and see if one of them is useful to you.

## Unknown command handling

Not bad! However, some confused users might try to send commands to the bot that it doesn't understand, so you can use a `MessageHandler` with a `command` filter to reply to all commands that were not recognized by the previous handlers.

```r
unknown <- function(bot, update){
	bot$sendMessage(chat_id = update$message$chat_id,
                        text = "Sorry, I didn't understand that command.")
}

unknown_handler <- MessageHandler(unknown, Filters$command)
dispatcher$add_handler(unknown_handler)
```

## Stopping the Bot

If you're done playing around, you can stop the Bot either by using the the `interrupt R` command in the session menu (in *RStudio* you can press the `STOP` button) or by calling the `updater$stop_polling()` method. Below we will define a command that uses this method:

```r
# Replace the original line with
updater <<- Updater(token = 'TOKEN')

...

# Example of a 'kill' command
kill <- function(bot, update){
  bot$sendMessage(chat_id = update$message$chat_id,
                  text = "Bye!")
  bot$clean_updates()
  updater$stop_polling()
}

kill_handler <- CommandHandler('kill', kill)
dispatcher$add_handler(kill_handler)
```

Now you can send the command `/kill` from Telegram to stop the Bot.

**Note:** With the [*superassignment* operator `<<-`](https://stat.ethz.ch/pipermail/r-help/2011-April/275905.html) we assign the `updater` in the enclosing environment so to call it from inside the `kill` function.

That's it for now! With this you may have the first guidelines to develop your R bot!

# Want more?

If you want to learn more about Telegram Bots with R, you can look at these resources:
- Package `telegram.ext` [GitHub Repo](https://github.com/ebeneditos/telegram.ext) or its [Wiki](https://github.com/ebeneditos/telegram.ext/wiki) to look at all methods and features available.
- Telegram's documentation [Bots: An introduction for developers](http://core.telegram.org/bots) and [Telegram Bot API](http://core.telegram.org/bots/api) to familiarize with the API.

# Attribution

This tutorial is adapted from [`python-telegram-bot` Wiki](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Extensions-–-Your-first-Bot).