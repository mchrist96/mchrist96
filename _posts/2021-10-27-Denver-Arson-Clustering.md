# Denver Arson Clustering
## Intro
I was recently informed that the city of Denver has a data analytics department that published a comprehensive data set of all crimes that have been reported in Denver over the past 5 years. I thought that this would provide an interesting opportunity to work with geospatial data in the form of coordinates (longitude and latitude) that are included for each crime entry. There are many clustering methods in existence, one of the most common is KMeans clustering which uses the square of Euclidean distance d(p,q) shown below.

![image](https://user-images.githubusercontent.com/92491074/138955248-71fe26a9-2a53-44a1-b3ed-b9bdbf7e1281.png)

The issue with Euclidean distance is it is meant for measuring distance on a flat plane and the globe is not flat. In order to take into account that our data exists on a globe I decided to also use another clustering method as well called DBSCAN which has the capability to measure distance (d) using the haversine shown below. 

![image](https://user-images.githubusercontent.com/92491074/138955363-23402f0d-9216-4104-94a3-6067b2f7a2ca.png)

A benefit of using the haversine to compute distance on a sphere, in particular in our case, is that it is numerically better-conditioned for small distances much like that of a city or state as opposed to large countries or continents. There is still the slight issue that using the haversine assumes a perfect sphere and the globe is not a perfect sphere but an oblate spheroid so there is still slight error with distance calculation using this method but that mainly applies to antipodal points, of which there are none in our data. Thus, there is no need to use the alternative Vincenty Formula that takes accurate measuring of antipodal points into account.

---

## Data Acquisition
To acquire the data, all one needs to do is go to `https://www.denvergov.org/opendata/dataset/city-and-county-of-denver-crime` and at the bottom of the page there are a handful of options for different file types. These include shape and esri geodata files that can be imported directly into software such as QGIS. I will QGIS later but for now I downloaded the csv file type and imported this into SQL. For information on how to do this check out my previous post on Spotify data analysis where I go in depth on this process. Once the data is in SQL we can inspect the data, which shows that there are `517638` rows and `19` columns. To add some focus to this project I decided to focus on crime with an `OFFENSE_CATEGORY_ID` of `Arson`. This is to cut down on the computation time as clustering methods can be quite time intensive when there are many points and I had just seen a movie about arson which motivated me to pick that crime in particular.

![Screenshot (84)](https://user-images.githubusercontent.com/92491074/139138213-01b623aa-3e24-4010-badd-b6dd5983fb26.png)

The next step is to connect Python, which I will be using to perform the different clustering methods, to SQL so we can query data from SQL directly in Python and assign the output to objects in Python. In my last post I did the same thing in R instead of Python using `RODBC` and this time I will do something very similar using `PYODBC`, the equivalent Python package. After installing and importing the PYODBC package I can execute the following code to connect to SQL and run a query for our data.

```python
# Establishing Connection To SQL Server For Data Queries
server = 'DESKTOP-00EIBV3'
database = 'Crime Data'
conn = pyodbc.connect('DRIVER={ODBC Driver 17 for SQL Server}; \
                      SERVER=' + server + '; \
                      DATABASE=' + database + ';\
                      Trusted_Connection=yes;')

# Querying Arson Data From SQL
sql = "Select * From [Denver Crime] Where OFFENSE_CATEGORY_ID = 'arson'"
Crime_Data = pandas.read_sql(sql, conn)

# Creating Arson Coordinate Data For Use In Clustering Functions
coordinates = Crime_Data[['GEO_LAT', 'GEO_LON']].to_numpy()
```

As you can see I queried all columns initially before later creating a coordinates object that had just the coordinate data needed as a numpy array. This is because initially there were other geographic indicators included in the data I considered using, such as address, before settling on longitude and latitude. Also, I was having fun inspecting other columns and the documentation that the Denver Police Department provides for this data uses the `OFFENSE CODE` column for reference. But after reading the documenetation and inspecting the data I no longer had need for the other columns and that is when I reduced to just coordinates.

---

## KMeans Clustering Method
To implement the KMeans Clustering Method in Python I will be using the Scikit-Learn library which is a machine learning library for Python that includes many clustering methods including both KMeans and DBSCAN. Additionally, it is built on both Numpy and Matplotlib, which I will be using as well. The KMeans algorithm clusters all given points, which in our data is `719`. This means that we need to identify the optimal number of cluster for the algorithm to use to assign all the coordinates we have. A common way to do this is using the Elbow Method which calculates the within-cluster sum of squares (WCSS) for various numbers of clusters (K) and plots them. When we look at the resulting plot the spot where the slope rapidly changes to approach horizontal is the optimal K (number of clusters). As we can see from the code and plot, included below, the optimal K is 4 and so that is the number of clusters we will specify for in our KMeans algorithm.

```python
# Running The Elbow Method for Determining Ideal Number of Clusters
wcss = []
for i in range(1, 10):
    kmeans = KMeans(n_clusters=i, init='random', algorithm='full')
    kmeans.fit(coordinates)
    wcss_iter = kmeans.inertia_
    wcss.append(wcss_iter)

number_clusters = range(1, 10)
plt.plot(number_clusters, wcss)
plt.title('The Elbow Method')
plt.xlabel('Number of Clusters')
plt.ylabel('WCSS')
plt.show()

```

![Elbow Plot](https://user-images.githubusercontent.com/92491074/139121765-b6c6ad83-9d44-4d44-b1b5-7730ed2a76ff.png)

In the code above you can see the initialization choice was set to `random` instead of the default `k-means ++`. This is because the default is meant to decrease convergence time but since our data is on the smaller side (cutting down from over 500,000 crimes to under 1,000 at the outset) there is little need for this and we can try random initialization. Additionally the algorithm was set to `full` as this is the traditional Expectation-Maximization style KMeans algorithm according to the documentation. The other algorithm option was `elkan` which is more efficient for data with well-defined clusters and as we will see later after running DBSCAN our data has a considerable amount of noise.

Now if we re-run our KMeans algorithm with the optimal settings we found above we can our resulting clusters and centroids. The code for the algorithm is shown below. You may notice that our first column is Latitude and our second column is Longitude which we have to assign in reverse order for plotting. This is because later using haversine distance it requires Latitude first before Longitude in our coordinates object and if we didn't take the time to explicitly assign x as longitude and y as latitude in our plot we would end up with backwards axes. Additionally, after plotting the KMeans model I export the results to Excel. This is for our use of QGIS later which I will touch on after addressing DBSCAN.

```python
#Performing KMeans Clustering
kmeans = KMeans(n_clusters=4, init='random', algorithm='full')
kmeans.fit(coordinates)

plt.scatter(coordinates[:, 1], coordinates[:, 0], c=kmeans.labels_)
plt.scatter(kmeans.cluster_centers_[:, 1], kmeans.cluster_centers_[:, 0], label='centroids', c='red')
plt.title('KMeans Clustering')
plt.xlabel('Longitude')
plt.ylabel('Latitude')
plt.legend()
plt.show()

#Exporting clusters and centroids to XLSX files to then resave as CSV to upload into QGIS as map layers
Kmeans_clusters = pandas.DataFrame(coordinates, columns=['Latitude', 'Longitude'])
Kmeans_clusters['Labels'] = kmeans.labels_
Kmeans_clusters.to_excel(r'C:\Users\Mitchell\Desktop\export_KMeans_clusters.xlsx', index=False, header=True)
cluster_centroids = pandas.DataFrame(kmeans.cluster_centers_, columns=['Latitude', 'Longitude'])
cluster_centroids.to_excel(r'C:\Users\Mitchell\Desktop\export_centroids.xlsx', index=False, header=True)

```

![KMeans Clustering](https://user-images.githubusercontent.com/92491074/139124515-970b820c-a8dc-4160-ad0b-bf45781dd992.png)

As we can see from the plot our 4 centroids are marked in red within each cluster. Additionally, as expected from KMeans every point is assigned to one of our four clusters. Next we will look at the other most famous clustering method in unsupervised machine learning, DBSCAN, to see how the clustering results change. The key differences between the two methods are that DBSCAN takes two crucial inputs, radius and minimum number of points, while KMeans only uses number of clusters. Further, DBSCAN is effective at detecting outliers and handling noisy data as it can detect high density clusters that are separated by areas of low density while KMeans is not capable of detecting outliers or noise and will assign those points to a cluster as it would any other points. Finally, KMeans uses Euclidean distance whereas DBSCAN can be assigned to use various distance calculation methods but in our case we will be using haversine.

---

## DBSCAN Clustering Method

To start our DBSCAN method we have to determine the optimal Epsilon and Min Number of Sample settings. Through trial and error of using various values I found that the best Epsilon, which is the maximum distance between two points for them to be considered in the same neighborhood, was equal to 1km. You will notice in the code below the value listed for Epsilon is `1/6371.008` this is due to our data having to be in radians for the haversine distance calculation and thus we must convert using the mean radius of the Earth which is `6371.008 km` according to NASA. Our next setting is Min Number of Samples which is the minimum number of points required to form a cluster. A general rule of thumb for this is to use 2 * the number of variables. In our case this would yield 4 as our optimal but if the data is very large, noisy or contains duplicate points it can be in your best interest to increase this number. Since our data is noisy I experimented with a few larger values and found 6 was the best choice for minimum number of samples. With these setting we find that almost exactly 10% of our points are filtered out as noise and not assigned to one of the 10 clusters. This is a drastically different result from the KMeans clustering method using Euclidean distance which had just 4 clusters of roughly equal size and which classified every point in our data. The code and resulting plots can be seen below. I included a plot at the end that is the DBSCAN clustering results with the noise removed so the clusters can be more easily seen.

```python
# Now for the DBSCAN model
dbscan_model = DBSCAN(eps=1/6371.008, metric='haversine', algorithm='ball_tree', min_samples=6)
dbscan_model.fit(np.radians(coordinates))

plt.scatter(coordinates[:, 1], coordinates[:, 0], c=dbscan_model.labels_)
plt.title('DBSCAN Clustering with noise')
plt.xlabel('Longitude')
plt.ylabel('Latitude')
plt.show()


#Creating and plotting just the clusters from our DBSCAN model
values = [index for index, element in enumerate(dbscan_model.labels_) if element == -1]

DBSCAN_clusters = pandas.DataFrame(coordinates, columns=['Latitude', 'Longitude'])
dbscan_model.labels_ = np.delete(dbscan_model.labels_, values)

for i in range(len(DBSCAN_clusters)-1, -1, -1):
    if i in values:
        DBSCAN_clusters = DBSCAN_clusters.drop(i, axis=0)


DBSCAN_clusters = DBSCAN_clusters[['Latitude', 'Longitude']].to_numpy()
plt.scatter(x=DBSCAN_clusters[:, 1], y=DBSCAN_clusters[:, 0], c=dbscan_model.labels_)
plt.title('DBSCAN Clustering with noise removed')
plt.xlabel('Longitude')
plt.ylabel('Latitude')
plt.show()

#Creating and plotting just the noise from our DBSCAN model
noise = Crime_Data[['GEO_LAT', 'GEO_LON']].to_numpy()
noise = pandas.DataFrame(coordinates, columns=['Latitude', 'Longitude'])
for i in range(len(noise)-1, -1, -1):
    if i not in values:
        noise = noise.drop(i, axis=0)

noise = noise[['Latitude', 'Longitude']].to_numpy()
plt.scatter(x=noise[:, 1], y=noise[:, 0])
plt.title('DBSCAN Noise')
plt.xlabel('Longitude')
plt.ylabel('Latitude')
plt.show()

#Exporting clusters and noise to XLSX files to then resave as CSV and upload into QGIS as map layers
DBSCAN_clusters = pandas.DataFrame(DBSCAN_clusters, columns=['Latitude', 'Longitude'])
DBSCAN_clusters['Labels'] = dbscan_model.labels_
DBSCAN_clusters.to_excel(r'C:\Users\Mitchell\Desktop\export_dbscan_clusters.xlsx', index=False, header=True)

noise = pandas.DataFrame(noise, columns=['Latitude', 'Longitude'])
noise.to_excel(r'C:\Users\Mitchell\Desktop\export_noise.xlsx', index=False, header=True)
```

![DBSCAN with noise](https://user-images.githubusercontent.com/92491074/139136909-43b00dbe-fd2d-44ee-a3fe-2ae6d8d047d1.png)

![DBSCAN without nosie](https://user-images.githubusercontent.com/92491074/139136935-7c17ac47-6ccb-42d8-a0eb-7d1220b75f77.png)

![DBSCAN noise](https://user-images.githubusercontent.com/92491074/139137712-f7a916af-74e2-4a26-94ce-802cd975ff82.png)

The code above includes more than just running the DBSCAN model. I exported some of my objects to Excel files in order to create visual layers in QGIS that are overlaid on a map of Denver. To create the separate layers I did some filtering of the data using for loops to remove coordinates based on their corresponding DBSCAN labels to get just the clusters and also data that was just noise. This way they can be uploaded as separate layers and toggled on and off at will on the QGIS overlay.

---
## QGIS Representation of KMeans and DBSCAN Methods 

QGIS is a free open-source cross-platform desktop geographic information system application that supports viewing, editing and analysis of geospatial data such as our coordinate data. This allows us to upload our data to an actual map of Denver and see how it looks as well as toggle off and on different layers of our data for each model. To do this we need to have our data in CSV file format. Our data has already been exported to XLSX previously so all we have to do is resave the files as CSV. Then we can open QGIS and go to the `Layer` tab, then `Add Layer`. From here we have multiple options for adding layers and we want to select `Add Delimited Text Layer`. From here a menu, show below, opens up where we can select our file from our computer and in my case it will automatically read our data and assign the Longitude column to our x-axis and Latitude to our y-axis as well as plot all of our points. I repeated this step for each of our 4 layers (2 for each model). Then for the two layers that involve multiple clusters (KMeans clusters and DBSCAN clusters) we go to that layer's properties menu. From here under the `Symbology` tab we can select the pulldown menu that is default set to Single Symbol and change this to Categorized and select the Labels column from our data as the value from which to determine categories. Then press Execute followed by Apply to distinguish the clusters. 

![Screenshot (104)](https://user-images.githubusercontent.com/92491074/139150301-fb13c8de-8dad-4f0a-8a4b-42680cdb3543.png)

Now we can toggle different layer combinations to see our model results in QGIS. To get a map background you can install and use the plugin `Quick Map Services` from the Plugins menu and then in my case I went with the OSM Standard map which is a world map. This will slow down the application so it is probably better in the future to pick a map that contains just the area your points cover. Below are the different maps I was able to generate using the model results.

### KMeans Model
![Screenshot (89)](https://user-images.githubusercontent.com/92491074/139150042-872f4fac-2200-454c-9984-636f252bdc57.png)


### DBSCAN Model
![Screenshot (90)](https://user-images.githubusercontent.com/92491074/139150066-97155311-6973-4ba2-9ef9-488764edc3c7.png)


### DBSCAN Clusters
![Screenshot (91)](https://user-images.githubusercontent.com/92491074/139150089-8b4d0050-b065-41a9-b05e-950d222f05d8.png)


### DBSCAN Noise
![Screenshot (92)](https://user-images.githubusercontent.com/92491074/139150106-0992f33b-cbe2-4ce5-9a61-bc423a308309.png)

