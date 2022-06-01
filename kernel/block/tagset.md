# tagset

tagset is an essencial struture of block layer. We will dive into the design of it in this article. Any comments are welcome! Especially pointing out my errors since I'm a kernel newbie. :)

## Structure 
Let's first have a overview of it.

## Initialization
Now let's see the init process of it. We've already know the last part of a nvme device initialization is nvme_dev_add() (nvme_reset_work()-->nvme_dev_add()). It basically initialize the tasgset structure.