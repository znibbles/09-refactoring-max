## Refactoring Max - When to Extract an Abstraction

One of my patreons challenged me to think, and talk about, how I handle large Max patches. I‘ve always felt that there is a massive tradeoff involved between avoiding huge monolithic patches and hiding complexity in too many levels of subpatches. While the latter practice certainly helps tidy up your programs, there is also an accompanying sense of losing clarity and control. So yes, it seems like there is a lot to talk about here, so I decided to start a series on Refactoring Max. 

This first episode will deal with when and how to extract code into an abstraction. First, an unambiguous sign that you should extract is when you encounter duplication, but it turns out that this is often more difficult to recognize than it sounds. 

Basically, I find myself encapsulating code in two types of abstractions:

1. Those that deal with data/signal transformation and/or processing („functional“)
2. Those dealing with manipulation, managing and holding of state/data („object oriented“)

While you‘ll find more opportunities for the first one - given that the whole range of audio processors fall under it, that‘s no surprise - we‘ll start thinking about the latter type. 

Whenever you find yourself repeating a _noun_ that you can combine with various _verbs_, chances are you have found an _object_ that can receive, and optionally even respond to, _messages_, a classic pattern in OO software design. Potentially one that even adheres to the _single responsibility principle_, which can serve as a guideline to find object _boundaries_. To keep your abstractions reusable, or composable, it’s pivotal that they don’t try to do too much, i.e. pull in functionality that doesn’t belong to their primary use case. Let’s look at some examples. 

## A Track Toggler

The first is a very simple one: suppose we have some external application which sends us OSC messages to select and deselect something, say, a track. It does so, however, on a button press, so every transition from 0 to 1 will mean a change of state, e.g. if the track is enabled or not. This is very common with MIDI controllers and the like. We will simulate this with a `[udpsend]` object and some message boxes.

Now when the message arrives, we first have to identify the OSC message, we can do this very easily with `[route]`. (I know there are more sophisticated options, that's not the point here.) Now we need it to listen for changes, so let's take a `[change]` object and connect the first outlet - the one that outputs the number we get fed in if it changes - to a `[sel 1]`. We know that the number coming in will either be a 0 or a 1, right? So if it is 1, we switch a toggle. Now we do this for every track, so we copy-paste. Done, right?

Then all of a sudden we encounter a new requirement: We need to be able to report the state of the `[toggle]` to an external caller. How do we do that? One option is a `[pattr]`. Now we need to insert it into every track. And here is the point where you should get suspicious:

1. we have duplication, and now we need to edit every single copy. Not good.
2. we have a message that we want to send ("what is your state?") and receive an answer to.

So it does seem as we'd want to make an abstraction. What do we call it? Well, continuing with our example, let's call it a `[track-toggler]`. We add a `[route]` object that will switch on incoming messages, and call the message `is_on?`. Let's try it out. Seems to work as expected.

Now all of a sudden, we get a new requirement. We should be able to set the initial value of the toggler to either 0 or 1. How can we approach that? Luckily, Max's abstractions have a special notation for that. A `#` followed by a 1, 2 etc. will give you the argument we passed to that abstraction. So if we say `[loadmess #1]`, this should send the first argument at creation time. We just need to _defer_ this to the end of the _low priority_ queue, so it won't fire until everything else is set up. Let's see. Works as advertised.

We now begin to see that with such an abstraction we can respond to feature requests quickly. What if we'd like to make the trigger threshold dynamic? What if we'd also like to store the history of changes? You get the point.

## A Folder Player

Maybe another quick example just to illustrate the principle. Here we have a patch that loads in all files in a folder (presumably they are all WAV files) and auto-populates a `[umenu]`. We can then send a number to the menu to select and output the prefixed path, which we will use to open an `[sfplay~]` object here. 

There's not too much going on here, but there are a few things to discuss:

- we obviously hold a (primitive) state in form of the entries of the `[umenu]`
- we send messages to it, such as the opening of the folder or the playback of files
- there are obvious extension points, such as the `[sfplay~]`'s channel count.

All of this seems very worthy of being encapsulated in an abstraction, but I would argue there is one point that needs to be considered very carefully: Would we like to include the opening of the folder dialog in the abstraction, send it a bang and then have it populate the menu? Everything would be in one place, right?

But then, what if we make multiple copies of this player and end up with a myriad of open dialogs we have to click through? Not so good. I would argue that including the folder choosing logic into the abstraction violates the _single responsibility principle_: After all, we're making a player, not a file system utility abstraction. So here's where I'd draw the boundary: in front of the `[prepend prefix]` object. That way, we can pass in folder paths from wherever they may come, be it a user input, an OSC command or whatever. 

There's even no need to extend the object's interface, because all messages are passed directly to the `[umenu]`. That said, I'd always advise you to be as explicit as possible when designing your code like this. Imagine weeks or months going by before you encounter this abstraction again, you'd be very glad if the messages you can send it gave you a hint at what you can do with it.

### track-toggler
```
----------begin_max5_patcher----------
670.3ocwV0riaBCD9L4o.QuRqr4mPZuT0mipUQNf2rdEXirMYy1U66dsGCIJ
Ij.cgr8hQdFim4aF+My71BufMh8TUf+O7+sum2aK77.QVAds68BpH6yKIJ3X
Ab5KhMOGD5Too60f3B5iTYo3kNErBPr4neEi5DpzuVRA4cR3MULdIUC2M9nP
Qi9boNQ5WqoN2MHv+gVU0Dc9SL910RZt1oEmD8MTneDJy9IcIrwHx+A6u79h
E1kvog4RAonhpT9eA2Cry9+gZbF7I66SB0aZzZAuGjs59frMD91ahtnDL.nk
tTaR25blSkF2h5yTqE7e1CziGIzi5E5QWMoFNPhMNFxq3THulMo7py58.tnI
kWQC48wtzWLpa8.fyEUUTt9DKx3ET.E3O.DAesuRQiDgnYjS5.e5pOKrqEa2
Zv1kfO4Pc3ZhjTQ0T4ZJmrwEHP2ENMyhrACNQYHWw5t0YfQuiH4FPBZZVhSQ
HTBBcAe23SZYeD8kShLDeSl93X6wHnqUB5Hau8WjTkVHauR7AwJxNZwZi2at
n0VbwLkvcc28NDytR1GzAA7YpPphV52WawaDWeTv0GxY+RxHkyQsUnsxIQbq
cTr+.G.a6kLzyS2PDw2kgHxex3e8QWSmbf5C7zzxWCOi19uFtZm9HE0a3Bt1
fRF+7wNg6yJ+zXnRzHy6bw1oO7OREJLTAFmnYlIUNdFK60+ZoowZnrQXH6rs
S1PKGggRlCCkNBCMGQtNm89mihGSrKZNLzHryouLqXE0BCQp8YtYx7TC2H0M
StgbzsaFbL7.NV5LD.vnOqWo3w734B63pqPpq2Qkp1CClvTE9YgztMKD1x3t
sPgPSuzcrtym.RHRScMsonViqEav9UtlVAUhBpj2vZGMyX42W7W..WRc7B
-----------end_max5_patcher-----------
```


### folder-player
```
----------begin_max5_patcher----------
918.3ocwW88aaBCD94j+JP7bVBFH+f9VkVq1j1lpV01dnpBY.2D2B1LaSZhp
59aemMPxRKzvVR6hTLgiy9tuO9tyNOzumcDeEQZach0UV858P+d8LlzF5Uce
O6L7p3Trz3lMibOO5V6AkORQVoLlE7BEwRpvBUnRfiuyRRTg2vSSHhZmoIFW
go+tw01jp0oDi4ZKrhLJKknLgyaqQH.O0ZoI05bRIBf0n5q00UtjiUwKnr4g
BRrpzKe2gNCrlhzinwd5KtfIqq0y3w980CC5HaTlAM.P+NBPTi.zo8rOvYnu
C7wEx5oA5j2yodbCpi4YYDlZmvRYIDCTbOl3z60Bm99GJNQ+C3zjvM.STGgo
SivD0pdc+5Tzz2Jr2RksxB0DgbXu3akQnZD8Rjh2vwZYgOvLSlLbh92S.Ih+
AUF2BzyEjbBKwhCiMwAtuNbvKR.ybGNdWrOa7qA1k2jmhW+KK2l.dWKGb+6Z
eKoyY3TcC7s+JByl2UEgqGZCqLNXKqTmr3kjjP..vbCwJkfFAaZIq3lJxAnK
rjFqJXTcfOAZDUUT2yF1NKke+7TdDNUQxx4+QIu9ohLLSEyE5jixY67zLdRI
QYVd6ZybAcNEfZJgMWsnDQHzTDJXXPfKZ1LfBTz36ja3fsyYSJT5M7ws1kbJ
vSsjI+r.mRUqaHYTzLhTIHvbKyjJEUoj5HWTAWugtpA0Uv+gpJOjoARf4z.y
ldPETEPa4hlpZlUaDqkQIB77c2CTfyHJhHjvvQk.2oUl.Wn3477hTrh7DxXe
LTc4GEzOUG8y1AEdSQHHeiWnuHgz.JUbcPyb.03v6wK00hCzCNtgKwrvj02Q
1mmdvhllFFQvhjV7c6KkR4fFieFGCaCvkKr9v6OYz2jDgbzshhHpLdwnuRjz
TJO7x0r3QgRXDVxub5kizm7Lq.jyi9DcIIXTDVDgCiKDXEN7rKrtPv0E9ire
osdd14G0Oulj9woe+L8SO8imedW14FTP0GgAE31nlxrJ1oT1SODtYU012UnI
4Eh35DtpXwZaGtDn7E5LTWxe0VgmUaZ4tFH3D6Vn8DnfiTbdS.jYQ1Ghzawe
3QxqKXBcLhjaWhztoCWn+KZsdlwianQMGZmCKznN7lz+XPunNfQuiQf5PbF+
r3T1LAmmuD5XV4rIDv9S2xMD8zAlaorxaMaEXKHKo096arfEPKMEzOqPTtsy
pYSrKmJbLFAqfVIXfH+X+eCqq.Ms
-----------end_max5_patcher-----------
```