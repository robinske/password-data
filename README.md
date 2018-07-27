# Password Data
CSV files containing password data used by the following:

Tutorial: https://www.twilio.com/blog/2018/06/analyzing-pwned-passwords-with-apache-spark.html

Talk: https://www.youtube.com/watch?v=3HjjxP0-uFE

Original data from [havibeenpwned](https://haveibeenpwned.com/Passwords).


## 500 Million Pwned Passwords 😱

The below is detailed information about how I got the data in this format. 

It's really boring. 

The [tutorial](https://www.twilio.com/blog/2018/06/analyzing-pwned-passwords-with-apache-spark.html) is far more interesting. 

Read on if you're curious. 

The raw data is available from Troy Hunt via torrent or file download here: https://haveibeenpwned.com/Passwords. Grab the Version 2 File that's ordered by prevalence and download it to your computer, unzipping it with your favorite program. You should end up with a single `.txt` file that's about 29G. 

Since we downloaded the ordered by prevalence data set, the most popular passwords will be at the top of the file. Let's trim our dataset so that we can spare our local machine's memory and get some insights more quickly.

We'll use the `head` bash command to take the first 100 million lines of data.

```
head -n 100000000 pwned-passwords-2.0.txt > pwned-passwords.txt
```

Remove the original file or move it to a different folder so that it doesn't interfere with the rest of our analysis.

We'll want to split this into multiple files since Spark works best with file sizes between 64 MB and 1 GB. I split the data into files of 2 million lines since that made each file about 120M.

```
split -l 2000000 pwned-passwords.txt
```

Next rename all of the files so that they contain a useful name and a suffix.

```
for file in $(ls); do mv "$file" "pwned-passwords-$file.txt"; done
```

## Install and Configure Zeppelin

*You can also use Docker - [that way is detailed here](https://www.twilio.com/blog/2018/06/analyzing-pwned-passwords-with-apache-spark.html#h.h2zqa8rvim79)*

We'll be using Apache Zeppelin to explore the data. Zeppelin is an open source project that allows you to create and run Spark applications from a local web application notebook. It's similar to Jupyter notebooks if you've worked with those in the past. For the purposes of getting familiar with Spark, we're only going to be looking at local data in this tutorial.

Follow along with the Zeppelin installation instructions and start up the server by typing `zeppelin-daemon.sh start` in your terminal. Zeppelin runs on port 8080, so navigate to http://localhost:8080 and you'll see the Zeppelin interface.

Note - the folder you start zeppelin from becomes your working file directory. 

## Adding a Library to Zeppelin
We're going to use the commons-codec library for its SHA1 hash function - don't roll your own crypto, kids! Zeppelin handles dependency management from its interpreter menu. Go to http://localhost:8080/#/interpreter and click `Create` in the upper right corner. We'll be adding our artifact at the bottom in the Dependencies section.

Enter `commons-codec:commons-codec:1.11` under `artifact` and hit `+` and `Save`.

## Making Useful Data
I downloaded the [10million passwords list from GitHub](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt). Organize your folder so that this is in a different path than the pwned passwords. This is so you can grab all the pwned passwords in a single glob match.

You should now have a folder of data that looks something like this:

```
$ tree
.
|____passwords.txt
|____pwnedpasswords
| |____pwned-passwords-xaa.txt
| |____pwned-passwords-xab.txt
| |____pwned-passwords-xac.txt
| |____pwned-passwords-xad.txt
...
```

Now we can write some code to join our Pwned Passwords with our plaintext passwords. First we need to split the pwned password data into a useful Dataset.

In a new cell, add the following code:

```
case class PasswordHash(pwnedhash: String, count: Int)

val pwnedpasswords = spark.read.textFile("./code/password-data/pwnedpasswords/*")
    .map { line =>
      val Array(hash, count) = line.trim.split(":")
      PasswordHash(hash, count.toInt)
    }
    
pwnedpasswords.printSchema
```

The `printSchema` function will show the structure of our data. 

Since the hash is a one way function, we will use our known passwords list to get their hash equivalents. This will rule out some of the pwned passwords, but for the sake of exploration we can afford to shrink our data size.

In a new cell add the following join code:

```
import org.apache.spark.sql.Dataset
import org.apache.spark.sql.functions._

def hash(password: String): String = {
  val hashed = org.apache.commons.codec.digest.DigestUtils.sha1Hex(password)
  hashed.toUpperCase
}

val plain: Dataset[String] = spark.read.textFile("./passwords.txt")

case class PasswordHash(password: String, hash: String)

val mapped = plain.map { pw =>
    PasswordHash(pw, hash(pw))
}

val passwords = mapped.join(pwnedpasswords, pwnedpasswords("pwnedhash") === mapped("hash"))
    .select(col"password", col"hash", col"count")

passwords.orderBy(col"count".desc).show()
```

This operation takes some time to run since we're processing gigabytes of data. In order to operate on the joined data later we can save it in a more useful form. 

```
passwords
    .write
    .option("header", "true")
    .csv("./csv-passwords")
```

Spark has built in capabilities for reading data in CSV, JSON, text, and more. Next time you want to read in this data, you can do so with the following:

```
val passwords = spark
    .read
    .option("header", "true")
    .option("inferSchema", "true")
    .csv("./csv-passwords/*")
```
