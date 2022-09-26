# virtiofsd

This article is about the backend of virtiofs: virtiofsd. And here I'll
pick up the rust version as an instance, while I'm also new to this languagethough. I'm going to talk about the big picture of virtiofsd as well as
diving deeply into the code to make you have a full sense of it.

### framework

Let's first look at a graph which shows the relationship of the main classes
 of virtiofsd.

![](./images/virtiofsd.drawio.svg)
