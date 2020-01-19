# Insecure Direct Object Reference (IDOR)

### Background:
It's a fairly typical pattern in web applications to see resources accessed by a unique identifier in the URL or POST parameters. Often, these are numeric or follow a predictable pattern. Consider the following example, on `innocentbank.com`:

`https://innocentbank.com/accounts/view?id=1337`

This instructs the application to return the account summary, for the account with id `1337`. In the backend, the application will do a query and return the information associated with that id. An "Insecure Direct Object Reference" occurs when the application does not verify that the current user is actually authorized to access the resource with that id. For example, if we're the user that owns account `1337`, but supply `1338` as the id - does the application return another user's account info? If so, we have an IDOR bug!

## Example

Take a look at the products section.

http://hack.jackmc.xyz/products/1

This is much like the example above, where we are accessing our product by the id `1`. In order to test if we have an IDOR via this parameter, we can simply change it to another product that we know exists (or test what happens when we increment the value).

In this case, attempting to access http://hack.jackmc.xyz/products/3 does not yield anything interesting:
> "Oops! This product doesn't exist, or you don't have access."

However, this action (the `products#show` action) is not the only action in which we interact directly with `Product` objects. When we edit our products, a similar pattern shows up:

http://hack.jackmc.xyz/products/1/edit

Manipulating this `id` parameter yields more interesting results! This is happening because we're doing an "unscoped find" here, which typically has the following pattern in Rails:

```ruby
# Looks up the product with this id (from the set of all products)
def set_product
  @product = Product.find_by_id(params[:id])
end
```

We should instead do lookups scoped to a given user, to only return resources that user owns:

```ruby
# Looks up the product with this id (from the set of products associated to this user)
@product = current_user.products.find_by_id(params[:id])
```

This now has no chance to return products the user doesn't own, since the search is scoped explicitly to the current user.


---
## Look for the other IDOR!
This app has at least one other IDOR, try and track it down!

### Questions:
#### Which page is vulnerable?
<details>
  <summary><b>Hint</b></summary>
  Try changing any numerical IDs you see - even ones that aren't
  in URLs!
  </details>

<details>
  <summary><b>Answer</b></summary>
  The endpoint at POST http://hack.jackmc.xyz/shops/3/edit has an
  IDOR vulnerability.
</details>

#### Try and exploit this bug!
<details>
  <summary><b>Hint</b></summary>
  Make a normal, legitimate request to update your own shop. See if you can manipulate it in a way that would act on a different shop.
</details>

<details>
  <summary><b>Answer</b></summary>
  Use dev tools to edit the <code>action</code> of the <code>form</code> tag here to reference a different id (right click and select "inspect" or "inspect element" to open this up).

  Alternatively, capture and replay the POST request to http://hack.jackmc.xyz/shops/1 that occurs on an update. Change the id value, and see if another shop is updated instead.

  How to do this:
  * If you are using a proxy like [Burp](https://portswigger.net/burp/communitydownload) or [OWASP ZAP](https://www.owasp.org/index.php/OWASP_Zed_Attack_Proxy_Project), you can capture and replay requests.
  * In dev tools in your browser it is also possible to edit and resend.
</details>


### Resources

[https://www.owasp.org/index.php/Testing_for_Insecure_Direct_Object_References_(OTG-AUTHZ-004)](https://www.owasp.org/index.php/Testing_for_Insecure_Direct_Object_References_(OTG-AUTHZ-004))

https://www.bugcrowd.com/how-to-find-idor-insecure-direct-object-reference-vulnerabilities-for-large-bounty-rewards/
