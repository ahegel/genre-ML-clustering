# Genre Clustering with Machine Learning
Using machine learning to reveal clusters of Goodreads shelves, supporting better descriptions of user behaviors and a more accurate understanding of genre online.

## Abstract
On Goodreads, users can create "shelves" to group similar books in their collection. But since there are over 15,000 shelves on Goodreads, to understand the types of shelves that exist and how people are using them, we need some way to simplify the data. By algorithmically grouping shelves into clusters using machine learning, we can automatically create collections of shelves that function similarly and reflect new types of literary genres that emerge on Goodreads. These new genres allow us to see patterns of similarity in shelving at a larger scale than we could through manual examination. By clustering shelves into genres, we can visualize genre on Goodreads in a way that reflects how people are actually using the site, rather than relying on pre-established definitions of genre.

The code for this project is [here](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py).

## Following along
The experiment outlined here walks through the code in [`genre_clustering.py`](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py). To run the experiment on your own data, replace `goodreadsshelves_sample.json` with your own Goodreads shelving data and then run [`genre_clustering.py`](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py) with your desired settings.

## Get bookshelf data
On Goodreads, each book has a list of "shelves" readers have placed the book on. We're going to cluster these shelves into genres by grouping shelves that have similar books on them. To start, we need data from Goodreads that shows each book, which shelves that book has been placed on, and how many times it was placed on that shelf, for as many books as possible. We can generate a file like this for historical Goodreads data using the Goodreads Shelves Wayback Spider in my [web-scrapers repository](https://github.com/ahegel/web-scrapers).  In [`genre_clustering.py`](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py), the function `get_wayback_shelf_data()` reads this information from a json file with shelving data from a [Goodreads shelf page](https://www.goodreads.com/work/shelves/4640799) ([example file](https://github.com/ahegel/genre-ML-clustering/blob/master/goodreadsshelves_sample.json)). One simple item looks like this:

```json
{
  "book_title": [
    "Life Code: The New Rules For Winning in the Real World"
  ],
  "people": [
    "22 users",
    "19 users",
    "15 users"
  ],
  "url": "https://web.archive.org/web/20160826202815/http://www.goodreads.com/book/show/17155775-life-code",
  "shelf": [
    "Self Help",
    "Psychology",
    "Nonfiction"
  ],
  "book_url": [
  ],
  "site": "goodreadsshelveswayback",
  "total_shelves": 1,
  "people_url": [
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/shelf/users/17155775-life-code?shelf=self-help",
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/shelf/users/17155775-life-code?shelf=psychology",
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/shelf/users/17155775-life-code?shelf=non-fiction"
  ],
  "shelf_url": [
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/genres/self-help",
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/genres/psychology",
    "https://web.archive.org/web/20160826202815/http://www.goodreads.com/genres/non-fiction"
  ]
}
```

[`genre_clustering.py`](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py) also includes a function to read data scraped directly from Goodreads: `get_shelf_data()`.

The function `get_wayback_shelf_data()` scrapes the json shelf data, cleans it, and returns a DataFrame for further analysis. You can call the function for any range of years in your dataset, and modify the parameters for max_ratio (the maximum ratio of books in the dataset that a shelf can contain to be included - use this to eliminate shelves that contain too many different books to be meaningful, like "to read"), min_books (the minimum number of books assigned to a shelf for that shelf to be included - use this to eliminate rarely-used shelves), and n (the number of shelves to include for each book - use this to limit to only a book's most popular shelves). The function returns `df`, the normalized DataFrame, and `df_raw`, the DataFrame with raw counts of the number of times people use a given shelf.

```python
years, max_ratio, min_books, n = [2013, 2014], .45, 0, 10
df, df_raw = get_wayback_shelf_data(shelf_json=shelf_file, max_ratio=max_ratio, min_books_using_shelf=min_books, n=n, years=years)
```

The result of the function, df, is a sparse DataFrame with each row corresponding to a book title and each column marking a shelf, where the values are the normalized number of people who have placed the book on that shelf:

book_title | animals | anthologies | anthropology
--- | --- | --- | ---
The Horse-Tamer | 0.13 | 0.0 | 0.0
The Return of the Sorcerer | 0.0 | 0.02 | 0.0
A House for Hermit Crab | 0.33 | 0.0 | 0.0
Partner to the Poor | 0.0 | 0.0 | 0.24

## Dimensionality reduction
Since the DataFrame is so high-dimensional (my full dataset has 2049 unique books and 1667 shelves), clustering won't be effective unless we reduce the dimensionality. There are several methods possible in `genre_clustering.py`, including PCA, multidimensional scaling, and isomap, but I found that t-SNE worked best for my data.

```python
# create distance/proximity matrix
dist = 1 - cosine_similarity(df.T)

# perform dimensionality reduction
x, y, names = reduce_dimensionality(df.T, dist, dim_type='tsne')
```

This plots the data in two dimensions (rather than 1667).

## Clustering
With this new, simpler representation of the data, we can now use machine learning to group shelves into clusters of shelves. [`genre_clustering.py`](https://github.com/ahegel/genre-ML-clustering/blob/master/genre_clustering.py) includes several possible clustering algorithms, including both hierarchical and k-means methods. For this application, since there is not a set number of clusters to find and clusters vary greatly in size, hierarchical clustering works best.

```
reduced_df = pd.DataFrame({'x': x.tolist(), 'y': y.tolist()}, index=names)
visualize_clusters_hierarchical(reduced_df, dist, x, y, names, method='ward')
```

Here's the result of hierarchical clustering with several prominent clusters highlighted:

![cluster scatterplot](/images/cluster_scatterplot.png)

This scatterplot shows the most commonly used shelves on Goodreads, placed based on how many books each shelf has in common with other shelves. If two shelves have a lot of books in common, meaning many users place the same book on both shelves, those shelves will appear close together. Groups of shelves that all share many books in common are shown as colored clusters. Let's zoom in on two of the clusters:

![cluster scatterplot](/images/sf_cluster.png)

The science fiction and fantasy cluster shows a tight grouping of shelves with little division between the two genres "science fiction" and "fantasy." Interestingly, a shelf for prospective mystery books as well as a cold war shelf seem closely related to the scifi-fantasy shelves.

![cluster scatterplot](/images/food_cluster.png)

The food cluster grouped books on food and cooking in a tight cluster. Impressively, the algorithm didn't use any information about the shelf names to group together these similarly-named shelves, but rather relied on readers using the shelves similarly to form the cluster. Adjacent shelves include "quick-read" and "read-non-fiction."

But there are too many shelves to see every cluster clearly. A dendrogram is helpful in this case. `genre_clustering.py` outputs a dendrogram of the results of the clustering algorithm (view it [here](https://github.com/ahegel/genre-ML-clustering/blob/master/images/hierarchical_clusters.png)).

Zooming in, we can see the World War I cluster much more clearly:

![WWII dendrogram](/images/dendrogram_wwi.png)

and a similar cluster for the Civil War:

![Civil War dendrogram](/images/dendrogram_civilwar.png)

It groups traditional genres surprisingly well:

![Biography dendrogram](/images/dendrogram_biography.png)

![Horror dendrogram](/images/dendrogram_horror.png)

![Science fiction dendrogram](/images/dendrogram_sf.png)

![Historical fiction dendrogram](/images/dendrogram_historicalfic.png)

![Cinema dendrogram](/images/dendrogram_cinema.png)

![Classics dendrogram](/images/dendrogram_classics.png)

We can also see less well-established genres:

![Chick lit dendrogram](/images/dendrogram_chicklit.png)

![Cookbooks dendrogram](/images/dendrogram_cookbooks.png)

![Vampires dendrogram](/images/dendrogram_vampires.png)

![Design dendrogram](/images/dendrogram_design.png)

![Web dendrogram](/images/dendrogram_web.png)

Stephen King is a genre of his own: 

![Stephen King dendrogram](/images/dendrogram_king.png)

And finally, some insight into the dissertation writing process:

![Dissertation dendrogram](/images/dendrogram_diss.png)