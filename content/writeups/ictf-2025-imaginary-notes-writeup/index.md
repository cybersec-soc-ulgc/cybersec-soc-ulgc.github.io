+++
title = 'ImaginaryCTF 2025 Web Writeup: imaginary-notes'
description = 'Writeup for ImaginaryCTF 2025 Web Challenge Imaginary-Notes'
date = 2025-09-24
author = 'David Craddock'
+++

**Author: David Craddock**

I solved the [ImaginaryCTF 2025 Web Challenge imaginary-notes](https://2025.imaginaryctf.org/Challenges).

As it was my first CTF (ever!) I made the mistake of not fully documenting what I was doing when I was doing it. Don't make my mistake! Make sure to take notes as you go through. I will attempt to piece together what I did in a summary version here.

The challenge text was
`"I made a new note taking app using Supabase! Its so secure, I put my flag as the password to the "admin" account. I even put my anonymous key somewhere in the site. The password database is called, "users".`

## Finding the API token and URL and deminifying the code

I used the Firefox developer console (press F12 in Firefox) to access the 'network' tab, and downloaded all the JavaScript assets that were included from the website.

I then collected these together, concatenated them together, and fed them to ChatGPT, asking it to de-obfuscate the code and explain it.

The code, typical to most JavaScript, was [minified](https://en.wikipedia.org/wiki/Minification_(programming)), which means it was deliberately rewritten by a minification library to both compress the file size of the resultant JavaScript files, but also to help [obfuscate](https://en.wikipedia.org/wiki/Obfuscation_(software)) it.

GenAI agents are very good at "deminifying" javascript, e.g. replacing the minified code with reasonable guesses at descriptive variable names and method/function names.

Once I had de-obfuscated the code, it was possible to trace through the execution of the code to a certain extent in my code editor by sight, and also to extract two important variables.

These two important variables were the API token and the URL to access the database. The URL was self-referential as the database was actually contained in a binary file coded in JavaScript that was also downloaded to the browser.

As the challenge text stated, the API key is all you need to access the database, even at root/admin level, which I thought was particularly 'kind' of the challenge writers!

## Collating the data and preparing the attack

Given the API key, the URL to access the database, and the knowledge that the password is in the 'users' table in the database, I was ready to start preparing an attack.

I used ChatGPT to give me an example snippet of code to connect to a [Supabase file-based database](https://supabase.com/docs/reference/javascript/introduction), given I was providing the API key and the URL.

I then copied and pasted this snippet text into the JavaScript console as part of the developer console in Firefox (F12). It acts as a JavaScript [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) and has access to all the in-browser data and objects that the browser would.

Despite looking at the documentation of the Supabase JavaScript database, initially I struggled getting the syntax correct, once connected, to execute a SQL `SELECT PASSWORD FROM USERS WHERE 'username' = 'admin'` type query.

So again, I used ChatGPT to help generate the correct syntax in JavaScript using the Supabase JavaScript library, making sure I included it correctly at the top of the copy and pasted attack script.

Once I had fully fleshed out the attack script with the help of the above, and tested it until it works, I was able to retrieve the `"password"` field from the admin user in the user table in the Supabase database, and that turned out to be the 'flag' token that I needed to complete the challenge.

## Conclusion

I found this CTF challenge extremely interesting and fun, and my only regret is that I did not take detailed notes as I went on. I hope to improve the quality of my write-ups in the future by knowing that this is the case, and making sure to make notes as I go along so that I can properly articulate how I did it.
