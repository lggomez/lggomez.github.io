{
    "title": "Scaffolding RSS feeds from zim notebooks",
    "description": "How to generate RSS feeds for zim notebooks",
    "summary": "How to generate RSS feeds for zim notebooks",
    "tags": ["Golang", "Post-mortem"],
    "date": "2021-01-23T15:10:02-03:00",
    "lastmod": "2021-01-23T15:10:02-03:00",
    "categories": ["Programming", "Backend", "Tutorial"],
    "type": "article",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "scaffolding_rss_feeds_from_zim_notebooks",
    "url": ""
}

As mentioned [after the fact](https://luisgg.me/About_RSS_feed.html), this site is built from a zim notebook and RSS generation is not one of the features included in zim as of today, so I immediately went into researching how to achieve this in the most straightforward but automatic way possible.

## Some notes on zim format 
Zim notebooks are file tree structures and tipically consist of 3 things:
* The zim notebook definition file, **notebook.zim**, which is located at the root directory. The format of this file is the same as .ini files, containing a main [Notebook] key with the notebook metadata and additional top level entries for plugins
* The wiki pages themselves, on txt format (at least by default). These files have an unique format with metadata present in the first 6 lines of the file
* The directories which contain subentries. These directories are accompanied by an equally named page, so that the directory is present in the wiki structure itself 

## Some notes on RSS 
The complete RSS structure and documentation can be read [here](https://www.xul.fr/en-xml-rss.html#structure), but sometimes a code snippet and some bullets say more than a thousand words:

```xml
<?xml version="1.0" ?>
<rss version="2.0">
<channel>
  <title>Main site title</title>
  <link>https://www.foo.com</link>
  <description>Site description</description>
  <managingEditor>mail@example.com (Author Name)</managingEditor>
  <pubDate>Wed, 23 Dec 2020 21:19:19 -0300</pubDate>
  <image>
      <url>https://www.foo.com/icon.gif</url>
      <link>https://www.foo.com/index.php</link>
  </image>
  <item>
      <title>Article 1</title>
      <link>https://www.foo.com/article1.html</link>
      <description>item description</description>
      <author>Author Name</author>
      <pubDate>Fri, 15 Jan 2021 20:34:28 -0300</pubDate>
  </item>
  <item>
      <title>Article 2</title>
      <link>https://www.foo.com/article2.html</link>
      <description>item description</description>
      <author>Author Name</author>
      <pubDate>Fri, 17 Jan 2021 21:35:29 -0300</pubDate>
  </item>
</channel>
</rss> 
```

* Format is XML (to dispel any remaining doubts)
* Top level element channel contains the entire feed
* Sub-elements are classified in two types:
	* Site metadata attributes
	* Item attributes, which are the articles themselves and their metadata
* Additionally, items can contain the entire article via the __<content>__ tag, but since we assume the content we're dealing with still hasn't been scaffolded to HTML the usage of this field is outside the scope of this project

## Putting stuff together in go 
The aim is to do the scaffolding in 3 steps:
1. Notebook traversal & metadata retrieval
2. RSS entity generation
3. XML file generation

For reading the notebook file and generating the RSS feed respectively, the following packages were used (and very useful):
* [github.com/gookit/ini/v2](http://github.com/gookit/ini/v2)
* [github.com/gorilla/feeds](http://github.com/gorilla/feeds)

Also, [github.com/jessevdk/go-flags](http://github.com/jessevdk/go-flags) always comes in handy for basic commandline parsing on tagged structs
The traversal becomes very simple by using the `filepath.Walk` function, which does a lexical BFS traversal of the directory (which makes it deterministic but may suffer performance-wise on dense directory structures, which is not our case). The main code is the following:

```go
func traverseAndParsePageMetadata(rootPath string, notebookPath string) zim.PageMetadataByCreationDate {
	pages := zim.PageMetadataByCreationDate{}
	walkErr := filepath.Walk(rootPath, func(path string, info os.FileInfo, err error) error {
		if path == notebookPath {
			return nil // ignore base notebook file
		}
		if err != nil {
			return err
		}
		if !info.Mode().IsRegular() {
			return nil
		}
		if !info.IsDir() {
			// TODO: Detect and ignore empty pages? (with empty content starting from line 7)
			pageMetadata, err := zim.ParsePage(path, info)
			if err != nil {
				return err
			}
			pages = append(pages, pageMetadata)
		}
		return nil
	})

	if walkErr != nil {
		panic(walkErr)
	}

	return pages
}
```

Then, RSS generation is similarilly simple:

```go
func createRSSFeed(pages zim.PageMetadataByCreationDate, rootPath string, zimNotebook zim.Notebook) (string, error) {
	// Create main feed element
	feed := &feeds.Feed{
		Title:       commandlineFlags.Title,
		Link:        &feeds.Link{Href: commandlineFlags.Link},
		Description: commandlineFlags.Description,
		Author:      &feeds.Author{Name: commandlineFlags.AuthorName, Email: commandlineFlags.AuthorEmail},
		Created:     pages[0].CreationDate.Add(-(60 * time.Minute)), // Use 1 hour prior to the first page creation as an arbitrary creation date
	}

	items := make([]*feeds.Item, 0, len(pages))
	for _, p := range pages {
		pageLink := p.PathToURL(rootPath, commandlineFlags.Link, zimNotebook.DefaultFileExtension)
		items = append(items, &feeds.Item{
			Title: p.Title,
			Link:  &feeds.Link{Href: pageLink},
			//Description: , TODO: no description available from page so this will be filled manually on generated file
			Author:  &feeds.Author{Name: commandlineFlags.AuthorName, Email: commandlineFlags.AuthorEmail},
			Created: p.CreationDate,
		})
	}

	feed.Items = items
	rss, err := feed.ToRss()
	if err != nil {
		log.Fatal(err)
	}

	return rss, nil
}
```

Finally, creating the file is just a matter of saving the return value of `createRSSFeed`...

```go
	// Write RSS feed file
	f, err := os.Create("rss.xml")
	if err != nil {
		log.Fatal(err)
	}
	defer f.Close()
	f.WriteString(rss)
```

... and that's it, now we have and RSS feed file. All that remains is adding any additional info that we want (like the article descriptions or the site image).

The commandline application's code is available at https://github.com/lggomez/go-zimrss for anyone interested.
