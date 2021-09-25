{
    "title": "Migrating the site to Hugo",
    "description": "The hard but rewarding road of website migration to Hugo",
    "tags": ["Golang", "Hugo", "Migration"],
    "date": "2021-09-24T00:45:06-03:00",
    "categories": ["Programming", "Web", "Tutorial"],
    "type": "article",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "migrating_the_site_to_hugo",
    "url": ""
}

If you've been following my earlier articles you probably know that the previous version of the site was scaffolded from zim notebooks (in this particular case, from a fork made by me including modifications on the code highlithging plugin, and then a manual [RSS generator from parsed notebooks](https://luisgg.me/en/scaffolding-rss-feeds-from-zim-notebooks/))

Also, if you're looking at this article you may have noticed I migrated from that modus operandi to a hugo static site.

## Rationale

The reasons are the following (in descending order):
* **Mobile support:** The older web template didn't have proper mobile support, and I didn't have the muscle to rewrite the template and the style to make it work
* **Easier maintenance:** Having to work on a personal fork on a tool for a purpose that is not the intended one (altough the HTML export feature of zim _is_ there other scenarios) is a workflow prone to errors and maintenance burden
* **Developer support:** Hugo is one of the most popular website generators as of today and aside from having an extended community it also counts with a large variety of templates to choose from, and a standard basis upon which all of these generate the web content (frontmatter + markdown)

## Some howto(s)

Hugo's website has a [quickstart tutorial](https://gohugo.io/getting-started/quick-start/). The main ideas being

1 - You set up an hugo configuration file
2 - You have to choose a theme, recommended as a github submodule (unless you want to do some heavy customization), but it is a part of your site's directory structure (since the content will be generated from there)
3 - Write your articles. Format is standard markdown [(and others)](https://gohugo.io/content-management/formats/), with a [frontmatter](https://gohugo.io/content-management/front-matter) header. Article location is [theme dependant](https://gohugo.io/getting-started/quick-start/#step-4-add-some-content)
4 - Preview your content locally with `hugo serve`
5 - Deploy your site (to be continued...)

In my case, the theme I ended up choosing is [zzo](https://zzo-docs.vercel.app/zzo), which provides a lot a features and a really nice user interface to leverage them

#### Other tasks done but that won't be detailed here
* Rewriting all RSS entry URLs
* Rewriting all zim entries as markdown hugo posts. Part of this was done with a modified version of the RSS scaffolder
* Rewriting all resource URLs to match their new locations
* Using a logo and reimplementing favicons
* Further CSS template customization (**TODO**)

## Deploying the site

As [told](https://gohugo.io/getting-started/quick-start/#step-7-build-static-pages) in the official guide, hugo generates the static HTML content based on the markdown posts, the theme and the configurations.

For the case of a github pages repo, a fully working site with minified assets will be generated with no further effort with the following command:

```
hugo -D --minify -d ./
```
