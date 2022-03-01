# Introduction

Lifetimes are Rust's most distinctive feature. They are what makes the language so valuable in the systems programming domain.
It's important to master lifetimes to use Rust effectively. Unfortunately, it's a very broad topic. 
To have a solid understanding of how lifetimes work you have to learn about them bit by bit from tons of 
official and unofficial Rust books, blogs, GitHub issues, source code comments, video materials, and your own mistakes. 
This book is my attempt to fix this situation. Here I'm trying to provide all knowledge I posses about lifetimes in an organized way. 
We will start from the `basics` chapter where we will explore how lifetime annotations affect the borrow checking. 
After finishing this chapter you should be able to annotate your code with a confidence and an understanding of what you're doing. 
Then we explore various topics that are nice to know about in certain situations. They're not as important as the `basics` chapter, 
you can revisit them when you need to.
