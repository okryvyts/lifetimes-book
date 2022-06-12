
# Русский военный корабль, иди нахуй 🇺🇦

# Introduction

Lifetimes are one of Rust's most distinctive feature. They are what makes the language so valuable in the systems programming domain,
and it's essential to master them to use Rust effectively. Unfortunately, it's a very broad topic: to have a solid understanding of how
lifetimes work you have to learn about them bit by bit from tons of official and unofficial Rust books, blogs, GitHub issues,
source code comments, video materials, and your own mistakes -- so this book is my attempt ot fix this situation.
Here I'm trying to provide all knowledge I possess about lifetimes in an organized way.
We start from the `basics` chapter where we explore how lifetime annotations affect the borrow checking,
after finishing this chapter you should be able to annotate your code with a confidence and an understanding of what you're doing.
Then we touch various topics that are nice to know about in certain situations and which are not as important as the `basics` chapter, so
you can revisit them when you need to.
