# **DISCLAIMER**: This is a showcase for EDUCATIONAL PURPOSES ONLY.

**Platform**: Portswigger
**Lab Name**: Finding and exploiting an unused API endpoint (API Testing Lab 2)
**Difficulty**: Practitioner
**Category**: API Hacking
**Date Completed**: 10-10-2025


**Objective**: To solve the lab, exploit a hidden API endpoint to buy a Lightweight l33t Leather Jacket.

# Solution -

The first thing I tried to do was to access a /api endpoint hoping to find some information. Unfortunately, the endpoint was hidden and I didn't have access to it.
So, I decided to investigate a little first. I looked at the javascript code to see where the api was being called, and I found this block of code:


```js
const getAddToCartForm = () => {
    return document.getElementById('addToCartForm');
};

const setPrice = (price) => {
    document.getElementById('price').innerText = price;
};

const showErrorMessage = (target, message) => {
    Container.RemoveAll();
    target.parentNode.insertBefore(Container.Error(message), target);
};

const showProductMessage = (target, message) => {
    Container.RemoveAll();
    target.parentNode.insertBefore(Container.ProductMessage(message), target);
};

const handleResponse = (target) => (response) => {
    if (response.error) {
        showErrorMessage(target, `Failed to get price: ${response.type}: ${response.error} (${response.code})`);
    } else {
        setPrice(response.price);
        if (response.message) {
            showProductMessage(target, response.message);
        }
    }
};

const loadPricing = (productId) => {
    const url = new URL(location);
    fetch(`//${url.host}/api/products/${encodeURIComponent(productId)}/price`)
        .then(res => res.json())
        .then(handleResponse(getAddToCartForm()));
};

window.onload = () => {
    const url = new URL(location);
    const productId = url.searchParams.get('productId');
    if (url.pathname.startsWith("/product") && productId != null) {
        loadPricing(productId);
    }
};
```

We immediately notice the loadPricing function, as it calls the API to fetch the price data. So I immediately went to this endpoint to see the response.
The response was a simple JSON with a price and a message. 

For the next step, I decided to do a bit more investigation and I opened up burpsuite. I took a bit to go around the application and understand how it worked.

There was generally two main API endpoints that I found. The one we mentioned earlier, in addition to a second one at /cart which handled adding/removing items from cart.

I played around with the cart functionality for a bit, though this kinda threw me off for some time. The cart endpoint had had a few parameters that looked like this:
POST -> cart/ (params: prodictId=${x}&quantity=${y},&redir=CART)

It took a POST HTTP Request with the specified parameters which were the product ID and the quantity (negative for removal, and positive for addition).
The cart had a bug where I could input a negative quantity of items even if the quantity available already reached zero. So for example, if I wanted to buy 1 jacket and added it to the cart.
Then I decided to send a request to remove 2 jackets (?quantity=-2) then the cart would still display the item with a quantity of -1, it only removed it when it reached exactly 0.

This threw me off for some time as I thought it was worth trying to try and influence the total price by trying, for example, to input a very large/low number so that it overflows.
I generally tried to find a way to have the negative quantity decrease or make the price negative as well (The assumption was that the total_price = single_item_price * quantity, without validating the quantity).
That proved fruitless unforutnately.

At this point, I thought I could somehow influence the API call that gets the price of the item, from the first api endpoint. Though I wasn't sure how I could do that (nor if it's even possible),
so I looked at a hint from a solution. The hint was that I could use the OPTIONS method instead of sending a GET Request to this endpoint.
The idea behind this is that the OPTIONS method would show what kinds of methods we could send, and so it did.

The options we had were these two: GET, and PATCH. A PATCH request is similar to a PUT request. They both can be used to modify a resource, the difference is the PUT request completely overwrites it.
The PATCH on the other hand, is usd to change specific parts of the resource.

In our case, the resource was this JSON:
```json
{
  "price": 1337,
  "message": "hello world!"
}
```
We now had the ability to change the price. We only had to specify a few things in the Request we were sending, which were that it was a PATCH request, the content-type which was application/json,
and the body of the request itself, which I set to:

```json
{
  "price": 0
}
```
And just like that, after editing the request in Burpsuite's Repeater and sending it. The price of the Jacket item was changed, and we could buy it for $0.

