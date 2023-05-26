---
layout: post
title:  "The Pokemon Gen 1 Poke Doll: Extremely simple, yet still broken"
date:   2023-05-26 17:42:38 +0200
categories: jekyll update
---
The first generation of Pokemon, that being Red/Green/Blue/Yellow, is very well known for not being the most well programmed games ever.
This obviously is not a surprise to anyone, but I still find it interesting to talk about why certain things are broken in this game by reading the games code and seeing why things are the way they are.

One of these things is surprisingly an Item that itself is extremely simple, yet still broken.

Let's talk about the Poke Doll! An Item that most of you probably have never used or maybe don't even remember existing at all in the game.

![Image of the Poke Doll inside the Inventory](/assets/gen1-pokedoll/pokedoll_inv1.png){:class="img-responsive"}{:width="50%"}

So, what even is the Poke Doll? What does it do and how is it broken?

In Gen 1 Pokemon, trying to flee from a wild Battle does a few checks to see if you can flee from the Battle or not. This itself has some math to it that I am not fully going into, but to give a rough run down:
1. Game checks if the Players Pokemon speed is higher than the Wild Pokemon speed, if so you can flee.
2. If it is lower, the game does some math to calculate your chance of being able to flee, if the math adds up, you can flee.
3. If the math does not add up, you fail to flee.

![Image showcasing Player failing to flee from Battle](/assets/gen1-pokedoll/battle_fleefail.png){:class="img-responsive"}{:width="50%"}

This is the most basic way I can currently describe the Pokemon fleeing system without going too in depth about it, but essentially, trying to flee from a Pokemon has a risk factor to it, you could always fail to flee and get attacked instead.

This is where the Poke Doll comes in! The Poke Doll is designed to be a single-use Battle Item that guarantees you a 100% flee chance in a Wild Pokemon Battle, essentially removing the risk factor regular fleeing has in Battle.

![Gif showcasing the Poke Doll letting you flee from a Battle](/assets/gen1-pokedoll/pokedoll_useexamp.gif){:class="img-responsive"}{:width="50%"}

So, now we know what the Poke Doll does! It's an Item you can use in Wild Battles to guarantee you a successful flee! So wait... how is it broken exactly?

Gen 1 Pokemon for Battles uses some WRAM entries to keep track of a bunch of important things, one of these is called `wBattleResult` and as the name of it says, it keeps track of the Battle Result state.

`wBattleResult` consists of 3 possible states:
* `$00`: You either just started the Battle OR you won the Battle!
* `$01`: You lose the Battle!
* `$02`: The Battle ended as a draw! (Fleeing from a Battle in this case counts as a draw.)

Whenever you start a Battle, `wBattleResult` gets always set to `$00` by a script called `init_battle_variables.asm` and here is where we get into some problems with the Poke Doll.

![Image of the Init Battle code](/assets/gen1-pokedoll/code_initbattle.png){:class="img-responsive"}{:width="100%"}

As mentioned earlier, whenever we flee from a Battle, `wBattleResult` gets set from `$00` to `$02` to tell the game that the Battle ended as a draw. This makes sense and you would expect the Poke Doll to do the same, right?

Well it turns out it actually does exactly not that! Now, I am not expecting most of you to be able to read what this code does, so let me break it down for you!

![Image of the Poke Doll Item code](/assets/gen1-pokedoll/code_usepokedoll.png){:class="img-responsive"}{:width="100%"}

But first, we should quickly go over this new WRAM entry called `wIsInBattle` as it is important to know what it actually is for in this code.

`wIsInBattle` is another entry that keeps track of if you are in a Battle or not, and what kind of Battle you are in!

`wIsInBattle` consists of 4 possible states:
* `-1` = You lost the Battle!
* `0` = You are currently in no Battle!
* `1` = You are currently in a Wild Pokemon Battle!
* `2` = You are currently in a Trainer Battle!

Pretty self explanatory, now with this in mind, let's go back to the actual Poke Doll code.

First, the game checks what kind of Battle we are in by doing:
```
ld a, [wIsInBattle]
dec a
jp nz, ItemUseNotTime
```
Essentially what this does is this:

1. Load the value `wIsInBattle` into variable `a`.
2. Take the variable `a` and decrease it by `1`.
	* (For example, this would turn the Wild Battle value from `1` to `0`, and the Trailer Battle value from `2` to `1`.)
3. Check if the variable `a` is not `0`
	* If we are in a Wild Battle, the variable should be `0`, if we are in a Trainer Battle it should be `1`.
4. If the variable `a` is not `0`, that means we are NOT in a Wild Battle and the game does not let us use the Item.

So pretty much, this first part is just a check to make sure we aren't trying to use the Poke Doll to flee from a Trainer Fight.

![Gif showcasing that you can't use the Poke Doll in Trainer Fight](/assets/gen1-pokedoll/battle_failpokedolluse.gif){:class="img-responsive"}{:width="50%"}

Now, if we are in fact in a Wild Pokemon Battle, the game will continue to run the following code:
```
ld a, $01
ld [wEscapedFromBattle], a
jp PrintItemUseTextAndRemoveItem
```
This part is pretty simple:

1. Set variable `a` to `$01`.
	* Pretty much, pretend `$01` in this case is setting `a` to `True`.
2. Set `wEscapedFromBattle` to `a`.
	* Aka, set `wEscapedFromBattle` to `True`.
3. Tell the Player that they used the Poke Doll Item and remove it from their Inventory.

Now at this point, in the core Battle system code, the game will detect that `wEscapedFromBattle` is now set to `$01`, which means it will simply just exit the Battle, letting us simply flee from any Wild Battle without any risk factors!

![Gif showcasing the Poke Doll letting you flee from a Battle](/assets/gen1-pokedoll/pokedoll_useexamp.gif){:class="img-responsive"}{:width="50%"}

Now, do you notice something weird about the code? The game never sets `wBattleResult` from `$00` to `$02`!!!

![Image of the Poke Doll Item code](/assets/gen1-pokedoll/code_usepokedoll.png){:class="img-responsive"}{:width="100%"}

This means that every single time we use the Poke Doll, the game considers the Battle as won instead of it being a draw, meaning this code is actually bugged.

Now this itself does not really sound like a problem, we can't use it in Trainer Battles to skip the fight as the game does not allow using the Item in Trainer Battles and using it in Wild Battles does not gain us anything either as we never defeat the Pokemon itself, so we don't get anything like XP.

So, this Bug means nothing, right? Well here is where it gets interesting! Using the Poke Doll, we CAN skip an entire chunk of the game because of this code oversight!

In case you don't remember, in Gen 1 there is the "Pokemon Tower" which is located in Lavender Town, which is a really important part of the game as you need to go there for a required Item called the "Poke Flute" to be able to access the rest of the games areas.

However, the Pokemon Tower can't be fully beaten at first due to you lacking an Item called the "Silph Scope". Without this Item, a Ghost that you can't attack will block the path, letting you not pass until you obtain said required Item.

![Gif of the Ghost Pokemon Battle](/assets/gen1-pokedoll/battle_ghost.gif){:class="img-responsive"}{:width="50%"}

Essentially what the "Silph Scope" does is let you identify what Pokemon you are actually fighting, revealing for example that the Ghost blocking your path is actually a Marowak. After it is identified, the game actually lets you fight the Pokemon which is required as you need to win against the Marowak to get passed the blocked path.

![Image showcasing that the Ghost is actually a Marowak](/assets/gen1-pokedoll/battle_ghostexamp.png){:class="img-responsive"}{:width="50%"}

Now to get the "Silph Scope", you need to get through this entire area filled with Team Rocket members hidden inside the Celadon Game Corner in Celadon City. This area itself is a bit long, has a lot of Trainers and is just kinda annoying.

But why would you wanna do all of that when we can just use the Poke Doll to skip all of this? Because as it turns out, this Ghost event just checks if `wBattleResult` is set to `$00` and if it is, it counts as beating the Marowak, letting you past the blocked path,
and as we learned earlier, the Poke Doll NEVER sets `wBattleResult` to `$02`.

So, because of this tiny oversight, we can skip the entire Team Rocket Hideout segment of the game, completely skipping getting the "Silph Scope" by simply using the Poke Doll!

![Gif that showcases skipping the Ghost enemy using the Poke Doll](/assets/gen1-pokedoll/battle_ghostskip.gif){:class="img-responsive"}{:width="50%"}

This could have been fixed very easily too! All it takes is add just three extra lines to the Poke Doll Item code and the issue is completely solved!

![Image of the Poke Doll code fixed up](/assets/gen1-pokedoll/code_pokedollfixexamp.png){:class="img-responsive"}{:width="100%"}

Below is said fixed code in action:

![Gif that showcases the Ghost enemy no longer being able to be skipped with the new code fix](/assets/gen1-pokedoll/battle_fixedcodeexamp.gif){:class="img-responsive"}{:width="50%"}

And there you have it! A very small oversight in the Gen 1 Pokemon code that straight up just lets you skip an entire part of the game, and the fix was super simple too!
I guess this is what we call a "Game Freak" Moment, huh.

Oh well, that's Game Freak for ya! Hope you learned something new today about this silly game! Will probably write more stuff soon!
