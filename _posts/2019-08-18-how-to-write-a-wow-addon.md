---
layout: post
title:  "How to Write a WoW Addon"
---

So you want to write a WoW addon? Well, you better get used to crappy docs, old websites, and skimpy examples. The fact that it's all community driven and nothing official from Blizzard doesn't make it any easier. I recently wrote my first addon for classic called [Track Sales](https://github.com/JoeGannon/TrackSales), so I know your pain. It's not all doom and gloom though, there's really only a few key concepts to understand and everything else falls into place. Though this guide isn't specific to retail or classic, please use the retail version to follow along. The version I wrote this with is `80200`.

First and foremost, if your struggling with your addon, the best advice I could give you is not to google, not to read the docs, not to read this post, but to read other addons! I've found hands down that has been the best way to learn how to do things. Generally, googling and reading docs are great tools but when developing `Track Sales` nothing was better than seeing how other people did things. Aim for smaller addons that will be easier to understand. Reading other people's code is always an excellent way to learn (for anything). You'll be amazed at the things you can learn just from reading someone else's code. Clone your favorite open source project and check it out someday!

## Hello Azeroth

We'll start with a Hello World example. In `_retail_/Interface/Addons` create a folder named `MyAddon`, inside create a lua file named `MyAddon.lua`. Place this statement `print("Hello Azeroth")` in the file. 

Next we need to tell WoW about our addon, this is done in the `toc` file. Create a new file named `MyAddon.toc` and add the below snippet. 

```
# You're client version is probably different than 80200
# In game run "/run print((select(4, GetBuildInfo())))" and use that for the interface number

## Interface: 80200
## Version: 1.0.0
## Title: My Addon 
## Notes: My First Addon
## Author: You 

MyAddon.lua
# additional lua files
```

This tells WoW what client version our addon supports (if the current version is newer than `80200` you'll see the "addon out of date" message), our addon name, and the files our addon needs. We'll come back to the `toc` file later but for now load up WoW. On the character select screen check to see `My Addon` is listed in the addon menu. If it isn't then double check that everything is correct. When you load into the game you should see "Hello Azeroth" appear in the chat. 

## Key Concepts

Building an addon can really be boiled down to these concepts. The rest is filling in the details of your addon. 

* [Events](#events)
* [Hooks](#hooks)
* [Saved Variables](#sv)
* [Ace](#ace) (3rd party helpers, optional)
* [Console Commands](#console)
* [UI](#ui) (sorry, I won't be covering this, see [Extended Character Stats](https://gist.github.com/JoeGannon/288ac79a9edadd6b639c138f6320b865) for ideas)
* [Testing](#testing)


<a name="events"></a>
## Events

Events are fired when various actions happen in the game world ie open mail, learned a new spell, etc. We can register to events 
with the code below. 

```lua
MyAddon = { }

local frame = CreateFrame("Frame")
frame:RegisterEvent("MAIL_SHOW")
frame:RegisterEvent("AUTOFOLLOW_BEGIN")

-- this syntax doesn't work in classic * 
frame:SetScript("OnEvent", function(this, event, ...)
    MyAddon[event](MyAddon, ...)
end)

function MyAddon:MAIL_SHOW()
    print("Mail Shown")
end

function MyAddon:AUTOFOLLOW_BEGIN(...)
    local player = ...
    print("Following "..player)
end
```

`MyAddon = { }` is essentially just a namespace. Lua doesn't actually have namespaces but it's the same concept as many languages such as `c#`, `java`, and `c++`. With this we can avoid using global variables. 

Unfortunately, I can't share much about `CreateFrame` other than "this is how you register events". You'll also use frames if you want to create a UI.

I have no idea what that crazy `SetScript` syntax is I just know that'll dispatch all events efficiently. It works by convention 
so you'll have to make sure the function name matches the event name. If you don't want to be restriced to that convention `AceEvent` can help.

When you open the mailbox and start following a player you should see the messages appear. You can find other events [here](https://wowwiki.fandom.com/wiki/Event_API).

<a name="hooks"></a>
## Hooks

Hooks are very similiar to events except that hooks fire before or after the `original` function (the built in WoW function). 
Hooks allow you to modify the parameters that get passed to the original function or if you wish, not call the original function at all...so you could really mess things up. "Secure Hooks" are the safe version to use. To create a hook we'll use `hooksecurefunc` which is a "safe post-hook", for your use case you may need pre or post hooks, [docs](https://wowwiki.fandom.com/wiki/Hooking_functions).

```lua
MyAddon = { }

hooksecurefunc("PickupContainerItem", function(...) MyAddon:PickupContainerItem(...) end)
hooksecurefunc("TakeInboxMoney", function(mailId) MyAddon:TakeInboxMoney(mailId) end)

function MyAddon:PickupContainerItem(container, slot)    
    _, _, _, _, _, _, link = GetContainerItemInfo(container, slot)

    print(link)
end

function MyAddon:TakeInboxMoney(mailId)   
    local _, _, sender, subject, money = GetInboxHeaderInfo(mailId)    
    
    print("Sender "..sender, "Subject "..subject, "Money "..money)
end
```

`GetContainerItemInfo` and `GetInboxHeaderInfo` are WoW apis, you can find other functions [here](http://wowprogramming.com/docs/api_categories.html). If you LEFT click an item in your inventory you should see it's item link printed. To test `TakeInboxMoney` send some gold to an alt and loot it from mail. Don't autoloot it! That would need the `AutoLootMailItem` hook. If it's not working, try disabling other addons and try again. There can be some intricacies to hooks, see the [notes](https://wowwiki.fandom.com/wiki/API_hooksecurefunc) section. Alternatively, try the `AceHook` package.

<a name="sv"></a>
## Saved Variables 

`SavedVariables` are values saved to disk that can be used between game sessions. The values are stored in the WTF folder (Warcraft Text Files not WTF) upon logout. There are two types of saved variables, `SavedVariables` and `SavedVariablesPerCharacter`. They are saved on a per account and per character basis respectively. Don't get tricked like I did, they're nothing but global variables! You just have to add the saved variables to the `toc` file.

```
## SavedVariables: AccountDB
## SavedVariablesPerCharacter: CharacterDB

# you can use multiple variables comma delimited
```

In your addon you'll usually want to initialize these the first time someone logs in with it. The `PLAYER_LOGIN` event
(`OnEnable` if using Ace) is probably the best place to do this because I found some of the WoW apis aren't ready at other times. Let's add a login counter to demonstrate. 

```lua
MyAddon = { }

local frame = CreateFrame("Frame")

-- trigger event with /reloadui or /rl
frame:RegisterEvent("PLAYER_LOGIN")

frame:SetScript("OnEvent", function(this, event, ...)
    MyAddon[event](MyAddon, ...)
end)

function MyAddon:PLAYER_LOGIN()

    self:SetDefaults()

    AccountDB.loginCount = AccountDB.loginCount + 1  
    CharacterDB.loginCount = CharacterDB.loginCount + 1

    print("You've logged in "..AccountDB.loginCount.." times")
    print(UnitName("Player").." logged in "..CharacterDB.loginCount.." times")
end

function MyAddon:SetDefaults()
    
    if not AccountDB then 

        AccountDB = {
            loginCount = 0
        }

        print("Global Initialized")
    end

    if not CharacterDB then 

        CharacterDB = {
            loginCount = 0
        }

        print("Character Initialized")
    end
end
```

<a name="ace"></a>
## Ace

If you've been reading about addons you've probably heard about [Ace](https://www.wowace.com/projects/ace3/pages/getting-started). At first I thought there was something special between WoW and Ace but nope, Ace is nothing but a 3rd party wrapper library over the WoW API. It's designed to make _some_ things simpler to do, and I say some because for your use case Ace might just get in the way (I ditched AceDB). Only use Ace if you really need it, some modules have a steep learning curve and it's easy to get bogged down in the details of Ace instead of focusing on the important parts &ndash; writing your addon! In this post I'll show how to use `AceHook` and `AceConsole`. `Ace Console` will be used for chat commands and also contains Ace's `Print` function.

Download Ace from the [site](https://www.wowace.com/projects/ace3) and extract the files somewhere. In your addon folder create a new folder named `libs`. Copy the folders `LibStub`, `AceAddon-3.0`, `AceConsole-3.0`, and `AceHook-3.0` into `libs`. Next create a new file named `embeds.xml` in your addon's root folder and add the following

```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.blizzard.com/wow/ui/ ..\FrameXML\UI.xsd">
	<Script file="libs\LibStub\LibStub.lua"/>
	<Include file="libs\AceAddon-3.0\AceAddon-3.0.xml"/>
	<Include file="libs\AceConsole-3.0\AceConsole-3.0.xml"/>
	<Include file="libs\AceHook-3.0\AceHook-3.0.xml"/>
</Ui>
```

Now we'll tell WoW about these libraries by adding the `embeds.xml` to the `toc` file 

```
## Interface: 80200
## Version: 1.0.0
## Title: My Addon 
## Notes: My First Addon
## Author: You

MyAddon.lua
embeds.xml
```

With Ace embedded now we can aceify the lua file

```lua
--the ace way of initializing addons, add the ace modules 
--you want to use as trailing parameters in :NewAddon()
MyAddon = LibStub("AceAddon-3.0"):NewAddon("MyAddon", "AceConsole-3.0", "AceHook-3.0")

function MyAddon:OnInitialize()				

    self:SecureHook("PickupContainerItem")

    local frame = CreateFrame("Frame")
    frame:RegisterEvent("MAIL_SHOW")
    frame:RegisterEvent("AUTOFOLLOW_BEGIN")

    frame:SetScript("OnEvent", function(this, event, ...)
        MyAddon[event](MyAddon, ...)
    end)

    self:Print("Hello from Ace Console")
end

function MyAddon:OnEnable()
	--initialize Saved Variables and other start up tasks
end

function MyAddon:PickupContainerItem(container, slot)    
    _, _, _, _, _, _, link = GetContainerItemInfo(container, slot)

    self:Print(link)
end

function MyAddon:MAIL_SHOW()
    print("Mail Shown")
end

function MyAddon:AUTOFOLLOW_BEGIN(...)
    local player = ...
    print("Following "..player)
end
```

Now your addon is bootstrapped with Ace. What changed is how the addon is initialized and the new `OnInitialize` and `OnEnable` methods. `OnInitialize` can be used to register events, create hooks, and anything else you might need to do on startup. 

If it all worked you should see `Hello from Ace Console` printed when you log in. Open the mailbox and follow random 
players, those should still work too. 

<a name="console"></a>
## Console Commands 

To use slash commands in your addon you can use [Ace Console](https://www.wowace.com/projects/ace3/pages/api/ace-console-3-0)
to register commands and parse arguments. 

```lua
MyAddon = LibStub("AceAddon-3.0"):NewAddon("MyAddon", "AceConsole-3.0")

function MyAddon:OnInitialize()				

	self:RegisterChatCommand("myaddon", "SlashCommands")
end

-- /myaddon 1 2 3
function MyAddon:SlashCommands(args)

    local arg1, arg2, arg3 = self:GetArgs(args, 3)	
    
    self:Print(arg1, arg2, arg3)
end
```

You can choose to handle arguments manually or use [Ace Config](https://www.wowace.com/projects/ace3/pages/getting-started#title-4), it all depends on your use case. I'd start with manual first. 


<a name="ui"></a>
## UI

Sorry, I didn't write a UI for `Track Sales` so I won't be covering it. You can check out an early version of [Extended Character Stats](https://gist.github.com/JoeGannon/288ac79a9edadd6b639c138f6320b865) for some general ideas, visit [curse](https://www.curseforge.com/wow/addons/extended-character-stats) for the latest. I advise you do this piece last and make sure the backend works first. Developing the UI and backend together is already hard enough no matter what environment you're in. 

<a name="testing"></a>
## Testing 

The hardest part about developing an addon is testing it. Sadly, there's no WoW sandbox or "Addon Dev Island" you can go to and have unlimited ability to test. Everything will have to be done manually in game. One nice thing is that you can call your addon's functions any time with `/run MyAddon:FunctionName()`. With this it's usually best to isolate the WoW api calls as best you can and have most of your addon's logic in functions that you can easily test with `/run`. You're also going to have to sprinkle `Print` statements everywhere. There's a couple [debug commands](https://wowwiki.fandom.com/wiki/Blizzard_DebugTools) that may be useful as well. To have code changes take effect run `/reloadui` or `/rl`.

## What Tripped Me Up

A couple things that weren't immediately obvious to me was how to call functions in other files and how to reference Ace modules in other files. This is done by variables that get passed in from WoW to every lua file. You can access them by placing `local addonName, ns = ...` at the top of any file. `addonName` will be a string containing your addon name and `ns` acts as a namespace. This [thread](https://www.wowinterface.com/forums/showthread.php?t=49349) details it well.

To see an example add two new files `MyAddonModule.lua` and `MyAddonHelper.lua` Don't forget to add them to the `toc` file!

```lua
-- MyAddon.lua

MyAddon = LibStub("AceAddon-3.0"):NewAddon("MyAddon")

function MyAddon:Test()				
    
    MyAddonModule:SayHello()
end

-- MyAddonModule.lua

MyAddonModule = MyAddon:NewModule("Module", "AceConsole-3.0")

local addonName, ns = ...

function MyAddonModule:SayHello()				

    self:Print("Hello from Module")
    ns:SayHelloHelper()
end

-- MyAddonHelper.lua

local addonName, ns = ...

function ns:SayHelloHelper()				

    print("Hello from Helper")
end
```

Test with `/run MyAddon:Test()`

With these key concepts you'll be able to build any addon. The tricky part is figuring out which [hooks](http://wowprogramming.com/docs/api_categories.html) and 
[events](https://wowwiki.fandom.com/wiki/Event_API) you need. `ctrl + f` is going to be your best friend here, becoming familiar with the naming convention helps as well. There's always forums like the [WoW addon reddit](https://www.reddit.com/r/wowaddons/) if you have questions. 

## Further Reading
* [Coding Tips](https://wowace.gamepedia.com/Coding_Tips)
* [Hooks Api](http://wowprogramming.com/docs/api_categories.html)
* [Events Api](https://wow.gamepedia.com/Events)
* [Classic Specific Info](https://wow.gamepedia.com/Porting_older_addons_to_Classic) *
* [Ace](https://www.wowace.com/projects/ace3/pages/getting-started)
