---
title: Simple pagination logic in pure Javascript
categories:
- Javascript
- Html
- Css
---

![alt](2021-11-22-Simple-Data-Pagination-in-Pure-Javascript/pagination.png)

Whilst many cool Javascript pagination packages out there, a lot of times what you might want is a simple function to handle pagination of certain items on a page. In such situations, installing a separate package to handle a simple pagination task might be overkill especially if you are particular and concerned about the portability and speed of execution of your code. In my opinion, installing a separate package to handle pagination of 200 items or less is definitely an overkill.

This guide is to help you understand the concept of pagination and how to implement a simple re-useable Javascript function that can be used to handle pagination of items of minimal sizes before being rendered on a page or returned as a response in an API request.

I will try to explain how you can easily build a simple pagination feature in a step-by-step approach for an easy understanding of the main logic.

At the bottom of this page is a jsFiddle live demo of this concept, feel free to skip all the explanations and move straight to the live demo if you can understand the code by yourself.

# Step 1
Given a list of ten Books, we want to display two books at a time i.e two books per page.
```js
var Books  = [
    { bookName : "book1"},
    { bookName : "book2"},
    { bookName : "book3"},
    { bookName : "book4"},
    { bookName : "book5"},
    { bookName : "book6"},
    { bookName : "book7"},
    { bookName : "book8"},
    { bookName : "book9"},
    { bookName : "book10"},    
];
```
Get the numbers of pages derivable from the Books list
We will start by dividing the book list by the numbers of books we want per page in our case two
```js
let pageSize = 2,
// Math.ceil() function always rounds a number up to the next largest integer. we need to avoid float values 
const numPages = ()=> Math.ceil(Books.length / pageSize);
```

# Step 2
Implement handlers for navigating pages-- current-page, next-page and previous-page

```js
let currentPage = 1;
// if currentPage is less than the total number of available pages move to next page
const nextPage= ()=> {
    if (currentPage < numPages()) {
        currentPage++;
        changePage(currentPage);
    }
}

// if currentPage is greater than one return to previous page
const prevPage = ()=> {
    if (currentPage > 1) {
        currentPage--;
        changePage(currentPage);
    }
}
```

# Step 3 
implement the change page handler for changing pages
```js
const changePage =(page) => {

    // let's make sure the requested page value is not less than 1 
    // if it is, default  page number to 1
    if (page < 1) page = 1;

    // let's make sure the requested page value is not greater than the entire available pages 
    // if it is, default page number to total number of pages
    if (page > numPages()) page = numPages();

    pages= [];

    let startPage = (page-1) * pageSize // this gives us the index of the book to start the page
    let endPage = (page * pageSize) // this gives us the index of the book to end the page 

    for (let i = startPage; i < endPage && i < Books.length; i++) {
        pages.push(Books[i]); 
        return [pages];
    }
```

here, we iterating throgh the Booklist and inserting the books that falls within the startPage  and endPage 
inside the  pages list. then we retured the pages

And we are done!.. thank you 

# JsFiddle Live Demo
<iframe width="100%" height="300" src="//jsfiddle.net/majeedblog/8nax7zyv/46/embedded/js,html,css,result/dark/" allowfullscreen="allowfullscreen" allowpaymentrequest frameborder="0"></iframe>
