## <center>Markdown Show</center>
---

### Text

&emsp;It's very easy to make some words **bold** and some words *italic* and some words ***italic and bold*** and some words <u>underscore</u> and some `inline codes` and some words ~~deleted~~ with Markdown. You can even [link to Google!](http://google.com)

<p align="left">left align</p>

<center>
center
</center>

<p align="right">right align</p>

<font color=red>color</font>

<font size=5>
size
</font>

<font color=yellow size=5>color and size</font>

<center><font color=green size=5>color and size and align</font></center>

noindent  
&ensp;one blank indent  
&emsp;two blanks indent  
bla&nbsp;&nbsp;&nbsp;&nbsp;nks  
bla&ensp;&ensp;&ensp;&ensp;nks  

### Lists

#### Ordered 

four blanks or one tab for indent

1. one

	1.1 one.one

	1.2 one.two

2. two

    2.1 two.one
    
    2.2 two.two

#### Unordered

two blanks or one tab for indent

* one 
* two 

- one
  - one.one
  - one.two
- two
	- two.one
	- two.two

#### Task 

- [x] This is a complete item
- [ ] This is an incomplete item

### Images

If you want to embed images, this is how you do it:

![](pictures/Snipaste_2021-06-03_12-01-25.png)

### Quotes

If you'd like to quote someone, use the > character before the line:

> author -- yeshen
> data -- 2021.6.3

### Codes

There are some c codes showed:

```c
int main(void)
{
	return 0;
}
```

### Tables

You can create tables by assembling a list of words and dividing them with hyphens `-` (for the first row), and then separating each column with a pipe `|`:

```
First Header | Second Header | Thrid Header
:----------- | :-----------: | -----------:
Content cell 1 | Content cell 2 | Content cell 3
first column | second column | thrid column
```

Would become:

First Header | Second Header | Thrid Header
:----------- | :-----------: | ------------:
Content cell 1 | Content cell 2 | Content cell 3
first column | second column | thrid column