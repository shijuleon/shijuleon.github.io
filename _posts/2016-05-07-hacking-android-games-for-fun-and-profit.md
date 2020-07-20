---
layout: post
title: Hacking Android games for fun and profit
date: 2016-05-07 22:46 
---

I have been noticing a recent trend of LAN based Android multiplayer games getting popular around me. It's a quite easy way to game with each other during class hours with comparitively low latency and with enough known players around, it's both fun and a sweet way to pass time during boring lectures. Current trend corresponds to Doody Army 2: Mini Militia but I don't know why it's so popular, so don't ask me. Everybody around me is playing it and I also tried joining a couple of times but got bored pretty quick. Pretty easy goal: Kill others by maximizing your chances with weapons. Well today, suddenly I got the temptation to look what's in it. Don't ask me why either.

Since apk packaging is no mystery and just a zip format archive, I used unzip to extract all its contents and looked into them. I then used dex2jar to convert the .dex files <!--more-->to .jar so I could use a Java decompiler to read the source. It threw a lot of errors and skipped a lot of classes. So, I tried reading the .dex files with jadx and it worked. Proguard had been used to obfuscate the code. Cocos2dx, fabric and a lot other ad analytics libraries were included. Cocos2D-X is a C++ port of the Cocos2D game engine API. Watch this, since this is the core of the whole game. 

Scrolling down the source code, I caught something interesting on the DA2Activity.class.

<code>
public static String getDirtyApps()<br>
  {<br>
    Object localObject1 = new String[5];<br>
    localObject1[0] = "org.sbtools.gamehack";<br>
    localObject1[1] = "cc.cz.madkite.freedom";<br>
    localObject1[2] = "com.forpda.lp";<br>
    localObject1[3] = "com.revealedtricks4u.minimilitiamods_revealedtricks4u.com";<br>
    localObject1[4] = "com.kingroot.kinguser";<br>
</code>

Looks like the developers have already came across some 'dirty apps'. It also checks for root access.

<code>public static boolean hasRootAccess()</code>

Most of the classes were obfuscated and I was not interested in figuring out how to deobfuscate that. But the sheer number of ad based libraries was really fishy. Talk about 'freemium'.

Inside the lib folder, there was <code>libcocos2dcpp.so</code> shared object file. This is where the actual application code of the game lies. The Java classes are just functions or wrappers invoking the compiled code inside .so file and thus to change the game functionality, we have to edit this particular file.

IDA Pro can read .so files and provide us with some pretty sweet information and lot of other features so I loaded the file in it. IDA was able to identify the functions and classes which provided a huge advantage. There were a lot of classes and a lot of code but I was interested in ammo, health and rocket boots. Even in these classes, the developers tried their best to confuse anyone attempting to read the code. Reading assembly is painful and reading game code especially without IDA, is very painful. Figuring out stuff from ARM assembly instructions might make you rage, so gather your patience.

To handle the networking part, it looked it Raknet was used. I don't have much experience on Android programming, so a lot of interfacing didn't make sense. Also due to thousand of lines of instructions, it was pretty distracting. Now, without setting a goal I knew I would get lost, so I decided to handle the weapons part first.

Figuring out weapons was easy. Each weapon was given a separate class (maybe to allow easy addition of weapons later). One of the first functions was <code>AK47::AK47(void)</code>. It seemed like a constructor and my initial thought of setting a constant high value was very shortlived. I don't really want to do that. Instead I selected the <code>AK47::triggerPull</code> function and tried to understand that. After the funtion prologue, there was an <code>SUBS</code> instruction with an immediate value of 1 which seemed to be executing everytime triggerPull is executed. Gotcha! Since weapons were switched frequently, I changed the <code>SUBS</code> to <code>ADDS</code> so that the ammo increases. You can also change the immediate value to 0, so nothing is subtracted. The reload function seemed to be executing when the ammo reaches 0, so it is never going to execute. I then used a hex editor to edit the opcode, saved the .so, modified the archive, signed with <code>jarsigner</code> and used adb to install the patched apk.

It took me some time to find an AK-47 in survival mode but the trick seemed to working when I did. Since there was a separate class for each weapon I had to do the same for every weapon.

Achievement unlocked: Infinite ammo

In most games, to avoid mods developers try to hide functions relating to health, so I was not surprised not able to find functions related to that. I tried keywords and but there were only a lot of functions relating to damage. Most of them were getter functions but the function <code>NetworkMessageDispatcher::updatePeerDamage()</code> seemed to be worth tinkering. This function seems to be updating the damage causes by one user to another. It loads the current status, subtracts and then updates. If nothing is subtracted then your health remains the same.

Achievement unlocked: Infinite health

As the same with health, I couldn't find the jetpack function with keywords. Instead after much searching, there was a <code>POWERUP</code> function which seemed to do that. Editing POWERUP functions seemed daunting so I searched for getter functions and found <code>SoldierHostController::hasPower</code> which seemed to indicate if the user had jetpack fuel left. Edited it such that it always returns 1 and boom infinite jetpack.

Achievement unlocked: Infinite jetpack

After making the changes by editing the opcode using a hex editor, I put back the .so file in the archive, signed it and then generated the apk. After testing it out in my device, it got boring, and having an app which collects too much data, ugh, I uninstalled the game.

Don't ask me for .apks, you are not getting it. Mod it yourself, dood.