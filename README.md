Macrochan is a gigantic archive of ancient 4chan images. It doesn't have image upload support, and has changed hands many times. Currently, it uses PHP as it's backend.

Because of it's 45,175+ image collection containing images that date from 4chan's early history, it would be 

This is an effort by the Bibliotheca Anonoma to archive Macrochan, using custom built Python scraping scripts to reduce strain on the server, and avoid the many pitfalls encountered by scraping an automatically generated database view.

It might also be possible to adapt this scraping script to other Image/Flash Collection websites, such as Dagobah, or even SWFChan.

## Scraping Feasibility Study

Usually, when we archive websites, we would use the ArchiveTeam's ArchiveBot, or Wget with WARC option enabled.

Unfortunately, due to Macrochan's tag structure design, conventional scraping is not the best option (and we'll probably end up in infinite loops clicking random.php or every single tag link in every page).

The three things we need to preserve are:

* The Filename. This way we can retrieve it. Also
* The Tags. They should probably be stored as image metadata, or maybe used on Hydrus. 
  * **The Tag Cloud.** - Also grab the nice tag cloud

## URL Structure

Macrochan is based on an image database written in PHP (used to be Python). Thus, you make queries to the database via the web interface to get what you want, though the responses are limited to 20 files per page.

### Get List of Every Image

To get a list of every single image, rather than just one tag, just go straight to `search.php` with no arguments:

    https://macrochan.org/search.php

This will give you a page browsing system that will show all 45,175 images in the collection, 20 files at a time. By clicking the `>>` (next page) button, the offset will increment by 20 each time.

    https://macrochan.org/search.php?&offset=0
    https://macrochan.org/search.php?&offset=20
    https://macrochan.org/search.php?&offset=40

As of 2015-03-22, since there are 45,175 images, the last page is:

    https://macrochan.org/search.php?&offset=45160

We simply need to make a list of numbers incremented by 20, all the way up to the final offset, and attach the URL `https://macrochan.org/search.php?&offset=` to the beginning. A for loop with a python script would work nicely.

We will also have to figure out the last number that we should put into the URL, which we call the `finalOffset`. This has to be a multiple of 20: so we simply use a modulus to reduce the number to the nearest multiple of 20. The algorithm for generating the `finalOffset` from the number of images is:

    finalOffset = numOfImages - (numOfImages % 20)
    # example
    finalOffset = 45175 - (45175 % 20) # = 45160

The amount of images that exist in the page can be found by checking the top of [the search query,](https://macrochan.org/search.php) or by parsing the following tag: 

```html
<br><center><div id=datebar>Showing 0-20 of 45175<a href="/search.php?&offset=20"> &gt;&gt; </a></div><BR></center>
```

Once we have a list of all the offset pages, we need to extract the image view URLs from it. We can use the handy Python libraries request and BeautifulSoup to scrape the HTML. Better yet, we can use the [RoboBrowser](https://github.com/jmcarp/robobrowser), which combines the two into a browser for robots.

The key is to extract any URL encased with the following tag:

```html
<a href='/view.php?u={{image-ID}}'>
```

### Obtaining image metadata

Once we have all the image ids, we now need to obtain two more things:

* **The full image URL with file extension.** This is because the image ID does not contain the file extension, which could be `.jpg`, `.jpeg`, `.png`, `.gif`, etc. Without the file extension, we won't be able to construct a working link to download the image itself.
* **The Image Tags.** One of Macrochan's most important aspects is it's tagging system. Each image is tagged with a certain topic word, which is then shown in the tag tree, and clickable to find all images with that specific tag. We need to preserve these as well.

First, an image URL has the following format:

    https://macrochan.org/view.php?u=<image-ID>
    # example
    https://macrochan.org/view.php?u=FCKU54O3VGXCPJMQ5RNSZ7P7ZRQBBL4F

However, this gives you the full HTML page, rather than just the image itself. You need to use a different URL structure to grab the full image itself, since the images are stored under two nested folders of 1st ID character and 2nd ID character.

    https://macrochan.org/images/<1st-char-of-ID>/<2nd-char-of-ID>/<image-ID>
    # example
    https://macrochan.org/images/F/C/FCKU54O3VGXCPJMQ5RNSZ7P7ZRQBBL4F.jpeg

In fact, we should probably replicate the practice of storing files under nested folders by 1st ID character and 2nd ID character (e.g. `/F/C/` for `FCKU54...`), since it is easy to generate and reduces the strain on file explorers of having to display over 40,000 images at once.

Then, we need to grab the image's tags. The image tags are stored as hyperlinks on `https://macrochan.org/view.php?u=<image-ID>`, and look like this:

    https://macrochan.org/search.php?&tag=Screenshots
    https://macrochan.org/search.php?&tag=Motivational+Poster

### Storing Image Metadata

We should also put the tag into a `.sqlite` database for convenient access, import, and queries. Obviously, since we are exporting from the Macrochan Image Database, we should use a similar format. 

This will also make it possible to generate views to easily show all the images that fit a certain tag, as well as add tags to certain types of images. You can't do that with flat files.

And for the archivist, databases make it possible to continue dumping where we left off. We can count the amount of `row_id` values already in the database, and just continue from there. Otherwise, we'd have to start over from the beginning, which often means hours of work wasted.

This will require an `images` table storing ImageID and file extensions, and a `tags` table storing TagName and TagURL. It will also require a linking table `taglink`, using foreign keys to match many ImageIDs to many Tags (many to many relationship). Here's the SQLite CREATE statements.

```python
# enable foreign key support
c.execute('''PRAGMA foreign_keys = ON''')
```

```sqlite
CREATE TABLE images (
  imageid text PRIMARY KEY,
  imageext text,
  imageurl text,
  imageview text
)
```

```sqlite
CREATE TABLE tags (
  tagname text PRIMARY KEY
)
```

```sqlite
CREATE TABLE taglink (
  imageid text,
  tagname text,
  FOREIGN KEY(imageid) REFERENCES images(imageid)
  FOREIGN KEY(tagname) REFERENCES tags(tagname)
)
```

## The Tag Tree

Now that we have all the image tags, we should also grab the Tag Tree.

    https://macrochan.org/tagTree.php

To obtain the tag tree, we should save the HTML page and convert it to Markdown "tag-tree.md" with this kind of structure: (JSON won't work due to it's complicated, and not relationally matched tag cloud)

```markdown
* #Type
  * Animated
  * Screencap
    * Anime
  * Screenshots
  * Ready to paste
  * Editable blanks
  * Shirt
  * Figurines
    * Lego
  * Broken
  * Wallpaper
  * Papercraft
```

We should probably make a pandoc bash script to do this in one go. Might as well grab [the news page](http://macrochan.org/news.php) as well.

### Tag Inheritance

As stated by the owner, one of the things he wanted to do was give tags inheritance. For example,

### Tag List Generation

Then, we grab all the tags from every image's JSON file, and make a list of all filenames that fit a certain tag. (we could grab the links from the site, but that requires offsets, takes time, and uses up server bandwidth)

## Step 2: Implementation

Now that we understand what to do, we have to do the hard work of actually implementing the script.

### 0-create-database.py

We need to create an accompanying `.sqlite` database to store source URL, file extensions, and all tags.

Make sure to enable foreign key support:

```python
# enable foreign key support
c.execute('''PRAGMA foreign_keys = ON''')
```

```sqlite
CREATE TABLE images (
  imageid text PRIMARY KEY,
  imageext text,
  imageurl text,
  imageview text
)
```

```sqlite
CREATE TABLE tags (
  tagname text PRIMARY KEY
)
```

```sqlite
CREATE TABLE taglink (
  imageid text,
  tagname text,
  FOREIGN KEY(imageid) REFERENCES images(imageid)
  FOREIGN KEY(tagname) REFERENCES tags(tagname)
)
```

### 1-search-query.py

The first thing to do is to obtain the image-ids and view links to all 45,175 images on the server. 

Of course, how did we figure out how many images on the server? To keep things simple, we just require the user to check `https://macrochan.org/search.php`, look at the amount of images, and provide it as an argument to the script.

Now, we could have automated this process of figuring out the amount of images, but what's the point? Macrochan doesn't have public image upload, and nobody seems to have uploaded an image since 2012.

The next step of the process is to put together the links to grab. by pumping offsets of 20 into the URL `https://macrochan.org/search.php?&offset=%s"` . As stated in the Feasibility Report, we will use an algorithm to calculate the `finalOffset` to pump into the URL.

```python
# calculate final offset with this algorithm:
#   finalOffset = numOfImages - (numOfImages % 20)
finaloffset = img_amt - (img_amt % offset)
```

First, we have to determine how many rows are currently in the SQLite database's `images` table (in case we are continuing where we left off). We do that with this SQL statement:

```python
c.execute("""SELECT MAX(rowid) FROM images""")
```

Once we have that, we use a typical for loop to make queries to the Macrochan Search system. 

```python
for i in range(0, finaloffset + 1, offset):
```

To obtain the image ids, we use [RoboBrowser](https://github.com/jmcarp/robobrowser), which combines Requests and BeautifulSoup into a browser for robot scrapers.

```python
		# set URL by offset
		url = site_url % str(i)

		# open the webpage
		browser.open(url)

		# beautifulsoup - find all <a href=> HTML tags that are views
		view_regex = re.compile(r"view")
		for anchor in browser.find_all('a'):
			view_tag = anchor.get('href', '/')
			if (re.search(view_regex, view_tag)):
				# obtain img_id from `?u=`
				img_id = parse_qs(urlparse(view_tag).query)['u'][0]
				# we don't know imageext or imageurl yet, so send NULL
				list = [img_id, None, None, view_url % img_id]
				c.execute('INSERT OR IGNORE INTO images (imageid, imageext, imageurl, imageview) VALUES (?,?,?,?)', list)

		# save changes to database when finished
		conn.commit()
```

Then, we dump the `img_ids` list into a SQLite database. This makes it possible to continue dumping where we left off by counting the amount of `row_id` values (so we don't have to start over from the beginning, which often means hours of work).

Finally, we put a delay between iterations to reduce strain on the server. Web Crawling can be a major drain on bandwidth (and thus operation costs), so you should do your best to show respect for the website you are lovingly archiving. 

By default, in this script we've set the delay to 5 seconds, so it takes several hours for the script to grab everything.

```python
delay = 5
# delay before next iteration
print("Waiting for " + str(delay) + " seconds...")
time.sleep(delay)
```

### 2-list-image-urls.py

Once we have all the image ids, we now need to obtain two more things:

* **The full image URL with file extension.** This is because the image ID does not contain the file extension, which could be `.jpg`, `.jpeg`, `.png`, `.gif`, etc. Without the file extension, we won't be able to construct a working link to download the image itself.
* **The Image Tags.** One of the most important search tools of Macrochan is it's tags on every image. We need to preserve these as well.

We can find these by parsing the URL `https://macrochan.org/view.php?u=<image-id>`, which contains the image url with the file extension, and the tags used on that image.

In order to make it possible to continue, from the SQLite database, we query the amount of rows as before, to set the end point. We also query the amount of `imageext` fields filled, to set the start point.

```python
# determine amount of rows in table, and calculate where to stop
# should be 0 for empty database
c.execute('SELECT COUNT(*) FROM images')
count = c.fetchall()
row_amt = count[0][0]
print("Table 'images' has {} rows.".format(row_amt))
stop = row_amt

# determine amount of rows in table with imageext, and calculate where to start
# should be 0 at beginning
c.execute('SELECT COUNT(*) FROM images WHERE imageext IS NOT NULL')
count = c.fetchall()
row_amt = count[0][0]
print("Starting at {} on table 'images'.".format(row_amt))
start = row_amt
```

First, we open the webpage using robobrowser:

```python
# open the webpage
browser.open(url)
```

I then use the following beautifulsoup function on the HTML to obtain all the image URLs, and extensions:

```python
# beautifulsoup - find first <img src=> HTML tag of main image to obtain file extension
img_url = macrochan_url + browser.find('img').get('src', '/')
img_ext = os.path.splitext(img_url)[1]
```

Finally, we need to grab all tags from the image view HTML and put it into a list. We use this BeautifulSoup loop, with regex to find tag URLs. Then we use `urlparse.parse_qs` to grab the tag name out of the URL.

```python
# beautifulsoup - find all <a href=> HTML tags that contain image tags
tags = []
tag_regex = re.compile(r"tags")
for anchor in browser.find_all('a'):
	this_tag = anchor.get('href', '/')
	if (re.search(tag_regex, this_tag)):
		# extract tag strings from tag url
		# http://macrochan.org/search.php?tags=Motivational+Poster
		tags.append(parse_qs(urlparse(this_tag).query)['tags'][0])
```

Now that we have the full image URL with extensions, we can use Requests to download the image in question. To reduce the strain on file managers, we will put the image into `images/<1st-char>/<2nd-char>/<image-id>.ext`. 

The `download_file()` function is also set to `clobber`, which will overwrite any existing file, just in case the script stopped in the middle of download. 

It also has a try/except that stops the script if download was unsuccessful, preventing the image ID from being added to the SQLite database. That way, SQLite insertion has an auxilary purpose of verification of successful image download.

```python
# download the images
# save images to folder `macrochan-dump-*/images/<1st-char>/<2nd-char>/<image-id>.ext`
img_filename = os.path.join(workdir, "images", img_id[:1], img_id[1:2], img_id, img_ext)
try:
	download_file(img_filename, img_url, clobber=True)
except requests.exceptions.RequestException as e:
	print("Unable to connect to Macrochan, restore your internet connection.")
	print("Run this script again to continue where you left off.")
	sys.exit(1)
```

Finally, we've now obtained the image file extension, the image itself, and all of it's tags. Now we need to put it into the database.

### Insert image metadata into Database

We insert the information into the SQLite database (per image).

```python
# OR IGNORE used in case we want to continue, and accidentally attempt to insert an entry we already put in
list = [img_id, img_ext, img_url, view_url]
c.execute('INSERT OR IGNORE INTO images (imageid, imageext, imageurl, imageview) VALUES (?,?,?,?)', list)
```

```python
# OR IGNORE used to avoid duplicating tags, since we will encounter them many times per image
for tag in tags:
  list = [tag]
  c.execute('INSERT OR IGNORE INTO tags VALUES (?)', list)
```

```python
# OR IGNORE used in case we want to continue, and accidentally attempt to insert an entry we already put in
for tag in tags:
  list = [img_id, tag]
  c.execute('INSERT OR IGNORE INTO taglink (imageid, tagname) VALUES (?,?)', list)
```

### How to Continue a Dump using SQLite

The SQLite database gives an additional advantage over flat files: it allows us to stop the dump, and continue later. This is critical when the connection is lost, or the computer runs out of power.

We use the following Python code to determine how many rows have already been committed successfully to a certain table, `images`.

```
# determine amount of rows in table, and calculate where to stop
# should be 0 for empty database
c.execute('SELECT COUNT(*) FROM images')
count = c.fetchall()
row_amt = count[0][0]
print("Table 'images' has {} rows.".format(row_amt))
firstoffset = row_amt - (row_amt % offset)
```

We can then use this for loop to start off where we left off. Or from the beginning, if there is nothing in the database.

```python
# Make search queries and place image IDs in list
for i in range(firstoffset, finaloffset + 1, offset):
```

This way, if the script stops for any reason, we can just run it again to continue, without needing to provide any arguments.

### Handy SQL Queries

Stuff that might be useful someday.

Note that images should use `NATURAL LEFT OUTER JOIN`, since they may or may not have tags, but should be displayed anyway. But with tags, they must have an image to be useful, so we only need `NATURAL JOIN`.

Show all images with their tag links, whether they have tags or not (value will be "" or NULL), ordered by rowid (which is the order of insertion):

```sqlite
SELECT imageid, tagname FROM images
	NATURAL LEFT OUTER JOIN taglink
	ORDER BY images.rowid;
```

Show all tags with their linked images. Ordered alphabetically.

```sqlite
SELECT tagname, imageid FROM tags
	NATURAL JOIN taglink
	ORDER BY tagname;
```

Find all tags applied to a single imageid (e.g. `4FFJSWB4T5MFYZSNFFJQG6W2W5XOVVDS`), whether they have tags or not:

```sqlite
SELECT imageid, tagname FROM images
	NATURAL LEFT OUTER JOIN taglink
	WHERE imageid = '4FFJSWB4T5MFYZSNFFJQG6W2W5XOVVDS'
	ORDER BY tagname;
```

Find all images that have a certain tag (e.g. `You're doing it wrong`). Sort alphabetically by `imageid`.

```sqlite
SELECT tagname, imageid FROM tags
	NATURAL JOIN taglink
	WHERE tagname="You're doing it wrong"
	ORDER BY imageid;
```