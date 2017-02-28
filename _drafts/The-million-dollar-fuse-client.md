---
layout: post
title: The six million dollar fuse client
---
"We can rebuild him. We have the technology. We can make him better than he was. Better, stronger, faster."

Gluster has a fuse client written in C. As an experiment over the last week my coworker and I took a shot at rewriting it in Rust. Rust has a low level fuse library so I saw this as a chance to learn more about how fuse works.  A few hours here and there over 4 days and we had something that worked! It wasn't perfect but some curious things came out of this. 

Most operations were slightly faster. I was playing with a small cluster on EC2 with a single client so this is by no means definitive but it is interesting. I have some suspicions but nothing conclusive yet.  It's possible that the C fuse client is faster under multi threaded operations. From reading the C code I see that all the fuse operations operate with callbacks. The Rust client is currently synchronous. Because multi threading is safe in Rust I've been curious if adding a thread pool might speed up small operations. 

Where are we at? I'd say about 80% of the file operations work as expected. I haven't made any attempt yet to optimize the code. Writing the client in Rust significantly reduces the surface area for memory leaks and undefined behavior.  My coworker Chris MacNaughton added an inode cache which really improved directory listing performance.  

Come check it out here: [Gluster fuse-rust client] () take it for a test drive and file any bugs you encounter. I'd really love any feedback and contributions. 