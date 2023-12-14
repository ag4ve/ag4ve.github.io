---
pin: true
layout: post
title: "Password Schema"                                       
date: 2012-10-20
categories:                                         
  - password
tags:
  - security
image:
  path: /static/img/Enigma_rotors_with_alphabet_rings.jpg
---

> UPDATE 2023-12-13: This information isn't entirely acurate. Wordlists are generally used which skews this math. I'm leaving it up as the elementary theory is correct but please don't base any decisions on the below.
{: .prompt-danger }

-----

A while ago a friend told me that a phrase was better to use for a password than a bunch of jumbled characters. For instance, using "This is a secure password." would "SD4#ac". I didn't believe him which started some email exchanges in which I basically proved myself wrong (with basic math). The following is what i found.

First, the technical side of things - if the services you use (facebook, twitter, comcast, banks, universities computers, work systems, etc) store your passwords correctly, when they get compromised, most passwords over 9 characters aren't easily compromised (this is a time sensitive figure, so in future years this won't be an accurate number). The way that see [1] how someone can try to retrieve at your passwords after the hash has gone public.

Note: that is as technical that I will get in this article. The remainder will be middle school math.

Now, for the math. Lets start by proving the math:

If you can have a password of length two, and you can only use three characters in this string (lets use a, b, and c), how many possibilities can you have?

aa, ab, ac, ba, bb, bc, ca, cb, cc

So, 9 possibilities. How do we mathematically get to this number? 3^2, or:

(possible characters)^(character length)

Note: This does assumes a constant of possible characters and a constant length. Though the formula allowing for less length is a bit harder to explain and doesn't add anything to the logic here).

There are 95 symbols on a qwerty keyboard (count them - 26 lower, 26 upper, 26 number row, 16 misc, and space). Most passwords can contain alternative characters, but most people won't use them. Most people use an 8 character password. I was informed today is nominal for a consumer desktop computer with a 1.6TB drive to crack an 8 character password - again, see [1] for explanation on how you can do this (and I broke my rule on being technical but figured it was worth it to point out that this is something you can do cheaply and is not just something governments do). Though, lets get past the technology and figure out how many tries it would take to brute force an 8 character password.

95^8 = 6634204312890625

But, lets assume you're better than most people and use 16 completely random keyboard characters:

95^16 = 44012666865176569775543212890625

This is a fairly large number that it takes a modern computer some time to brute force and a day to crack properly (think years, but again, see [1] below). So, why is using a random password the wrong way to do it? There's a few reasons:

1. it's hard to remember,

2. you're going to use less characters, so it'll be easier to crack,

3. because it's so hard to remember you're less likely to want to change it.

If however, you're one of those people that can memorize a deck of cards, has memorized Pi, 1^(1/2), or ln to 100 decimal places, you're probably using very strong passwords and can stop reading now - this isn't for you. For everyone else...

My recommendation - use a proper english sentence. Lets do the math and see why. Lets use this string for an example: "I've got this bank account that i'm trying to remember the password for."

This is quite a simple sentence, and it's 73 characters long. For the math, lets assume it's all lower case alphabetic characters. so:

26^73 = 1963606232436475854501753716263799429014060410113303954060845728432540419682760498206668083388875800576

That's a bit wrong because there are 3 non-alphanumeric characters - a space (' '), a apostrophe ('''), and a period ('.') and if I were writing a proper sentence I would've capitalized 'I' in "I've" and "I'm". But i figure this gives enough of a difference to show why you might want to change how you think about what a good password is. Maybe not - i could use a dictionary attack on a password and get at the password a bit quicker. Lets assume there are 50,000 english words most americans would use every day. if we do that, we do the math for 13 words in our phrase Again, the base is the number of things in our phrase or words to the power of length or the number of words. So:

50000^13 = 12207031250000000000000000000000000000000000000000000000000000

This isn't as long as doing a basic brute force attack (and you still have to compensate for the period and proper capitalization) but it's still orders of magnitude better than your random 8 character password and still easier to remember than S4$5bA^0 - right?

There is a slight caveat - if someone knows your password schema, it's not hard to narrow down the majority of the words you might use to the 10,000 (or less) most used and hope you didn't use a word with more than three syllables. So I would recommend using a proper noun in your sentence - "Niagra Falls was quite noisy in the summer", "A dark Guiness tasts good in the summer", "I live in the the common wealth of Virginia", "I'm buying the most awesome pair of Levis jeans. I can't wait for them to show up so I can wear them", etc. Speak a non-english language? You've just thwarted most dictionary attacks. Know chineese, or hebrew? You can make a pass phrase that'll be take generations to crack using current technology because of the character set (if you're password is accepted of course).

If you run into a system that limits password length - I give them hell on as many social networks as possible as there is *NO* reason to limit password length (see one way hashing algorithms for the reason why, take it as fact that there is no reason anyone should limit the length of the password you use). Other than that, i try to think of a password with as many long words and proper nouns as possible within the constraint. if a system places limits on characters (again, there's no reason for this) replace the character with something else - I have toreplace spaces with 0s often.

Most people recommend against writing passwords down. If you're argument is about making a long password vs writing it down - Write them down! Just don't write what they are for. If you use this method (of a sentence as a pass phrase), you should have a sentence describing the source (to make remembering it easier even if it is written down) that generally doesn't contain the sources name. Keep the list in your wallet - who cares if you loose your wallet, you didn't write the source of the password so you still have a separation of the lock from the key. And how often do you loose your wallet anyway? If you do, and you didn't write the source of your credentials by them, I think you can wait a little wile before paranoia gets you to change your pass phrases.

Lastly, what type of attack are you thwarting? When your bank, or power company, or a site like linkedin, or your employer is hacked and people put the password dump out on the internet (or just keep the list to themselves), people start going to work on that dump. If your password isn't one of the ones that gets pulled from the password dump you're password is safe.

We all know that most people use the same passwords fur multiple accounts, so the theory is if you have an account on one system that is hacked and the password is retrieved from a password dump, they'll be able to get data (or do other malicious things) from your other accounts. By making a password no one can grab with standard attacks against a password hash, you insure they can't login to your account from the place that was just hacked. You also insure that if you did use the same password on many accounts, you won't have to run around changing the password (if you even remember which sites you used that password on in the first place). Last, this schema allows you to easily remember different passwords on different sites so that none of these are issues for you.

1. How I'd go about cracking passwords:<br>
http://blog.thireus.com/cracking-story-how-i-cracked-over-122-million-sha1-and-md5-hashed-passwords

2. If you care about how to run through password hash quicker:<br>
http://www.pwcrack.com/rainbowtables.shtml

3. This xkcd was pointed out to me (though I think it is sorta wrong on the math): <br>
http://xkcd.com/936/

[ I gave this as a live demo talk at HacDC's Cryptoparty (http://www.cryptoparty.org/wiki/Washington,_DC) and I figured it would be good in print form ]
