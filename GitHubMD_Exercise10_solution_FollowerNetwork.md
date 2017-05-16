Solution to Exercise 10 - Which Swiss politician is the most popular online?
================
Dr. David Garcia
18-05-2017

**4.3 Extra: Repeat for follower network**

Repeat the analysis by retrieving the list of friends of each account, constructing the directed network of follower links between politicians. Keep the rate limitations in mind and that it will take some time. Compute the party assortativity of the follower network.

``` r
## load the politician list and look up their twitter accounts
tweeters <- read.csv("SwissPoliticians.csv",sep="\t",header=TRUE, stringsAsFactors=FALSE)
tweeters$screenName <- tweeters$twitterName

require(twitteR)
```

    ## Loading required package: twitteR

``` r
consumer_key <- 'X2QkLTfDYZJQee3eZIfs4PbfO'
consumer_secret <- 'HKeDcjgqlRkhXGg4W0FBNa46xN05IRERWHCnvcI3hYhOKHgexh'
access_token <- '747703282408296448-J5EVuVKvBkz7fG22LsOURiZPG8IefB3'
access_secret <- 'TClX1U2Tscj0QCCL5fGVWwhjjx4m0vJjtuFrlqvHcwkLI'

setup_twitter_oauth(consumer_key, consumer_secret, access_token, access_secret)
```

    ## [1] "Using direct authentication"

``` r
presentUsers <- lookupUsers(tweeters$screenName)
#presentUsers <- lookupUsers(tweeters$twitterName)
usersdf <- twListToDF(presentUsers)
#usersdf$party <- tweeters$party_short

library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:twitteR':
    ## 
    ##     id, location

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library(magrittr)
usersdf %>% filter(!protected) %>% select(id,screenName,followersCount,name) -> usersdf
usersdf$screenName <- tolower(usersdf$screenName)
tweeters$screenName <- tolower(tweeters$screenName)
inner_join(usersdf, tweeters) -> usersdf
```

    ## Joining, by = "screenName"

``` r
save(usersdf, file="usersdf_Follower_15052017.RData")
```

``` r
## define a function to handle the Twitter API rate limits
sleep.on.rate.limit <- function(sleep.time=0, resource=c("account", "friends", "followers", "users")) {
  rate.limit <- getCurRateLimitInfo(resource)
  if (any(rate.limit$remaining == 0)) {
  cur.time <- Sys.time()
  dtime <- max(c(0, difftime(rate.limit$reset[rate.limit$remaining==0], cur.time, units="secs")))
  sleep.time <- sleep.time + dtime
  message(paste("*** Sleeping for", round(dtime), "seconds ***"))
  Sys.sleep(dtime)
  }
  rate.limit <- getCurRateLimitInfo(resource)
  while (any(rate.limit$remaining == 0)) {
  sleep.time <- sleep.time + 1
  message("*** Sleeping an additional second ***")
  Sys.sleep(1)
  rate.limit <- getCurRateLimitInfo(resource)
  }
  return(sleep.time)
}
```

\#\#. most-time-consuming part, which takes about 6 hours

``` r
## retrieve the friend-follower arcs of the network
start.time <- Sys.time()
sleep.time <- 0

presentUsers <- lookupUsers(usersdf$id)

edgesDF <- data.frame(from=NULL, to=NULL)
for (idx in 1:length(presentUsers))
{
  sleep.time <- sleep.on.rate.limit(sleep.time)
  curFollowerIDs <- presentUsers[[idx]]$getFollowerIDs()
  curFollowerIDs <- curFollowerIDs[which(curFollowerIDs %in% usersdf$id)]
  
  newFollowerEdges <- data.frame( from=curFollowerIDs, to=rep(presentUsers[[idx]]$id,length(curFollowerIDs)) )

  edgesDF <- rbind(edgesDF,newFollowerEdges)
}

end.time <- Sys.time()
message(end.time - start.time)
message(paste("Slept for", sleep.time, "secs"))

edgesDF <- unique(edgesDF)
save(edgesDF, file = "FollowerEdges_15052017.RData")
```

``` r
## prepare color codes for the graph plot
library(igraph)
#load("usersdf_Follower_15052017.RData")
usersdf$party <- usersdf$party_short

usersdf %>% select(id,screenName,party) -> vertices

#Party color codes, to keep in handout
colCodes <- data.frame(party=c("AL","BDP","CVP","EDU","EVP","FDP","GLP","Green","SP",
                               "SVP","UP"), 
                       color = c("darkred","yellow","orange", "pink","darkblue",
                                 "lightblue","lightgreen","green","red","black","gray"))

colCodes$party <- as.character(colCodes$party)
colCodes$color <- as.character(colCodes$color)

inner_join(vertices,colCodes) -> vertices
vertices$name <- rep("",nrow(vertices))
```

``` r
## visualize the friend-follower network
load("FollowerEdges_15052017.RData")
edgesDF %>% filter(as.character(from)!=as.character(to)) -> edges

require(dplyr)
#which( (edges$from %in% vertices$id) & (edges$to %in% vertices$id) )
edges <- edges[ which( (edges$from %in% vertices$id) & (edges$to %in% vertices$id) ), ]
edges <- unique(edges)

g <- graph_from_data_frame(edges,vertices=vertices)

plot(g, vertex.label.color="black", vertex.label.cex=0.4, layout=layout.auto, 
     vertex.size=5, edge.curved=0.1, edge.width=1, edge.arrow.size=0.3)
```

![](GitHubMD_Exercise10_solution_FollowerNetwork_files/figure-markdown_github/unnamed-chunk-5-1.png)

``` r
# find out the politician with max in/out degrees
V(g)$screenName[degree(g,mode = "in")==max(degree(g,mode = "in"))]
```

    ## [1] "cwasi"

``` r
V(g)$screenName[degree(g,mode = "out")==max(degree(g,mode = "out"))]
```

    ## [1] "cwasi"    "felixzrh"

``` r
## tests on assortativity
V(g)$screenName[degree(g,mode = "in")==max(degree(g,mode = "in"))]
```

    ## [1] "cwasi"

``` r
V(g)$screenName[degree(g,mode = "out")==max(degree(g,mode = "out"))]
```

    ## [1] "cwasi"    "felixzrh"

``` r
ae <- assortativity_nominal(g, types=as.factor(V(g)$party))
ae
```

    ## [1] 0.3412593

``` r
a<-NULL
for (i in seq(1,10000))
  a[i]<-assortativity_nominal(g, types=sample(as.factor(V(g)$party)))

hist(a, xlim = range(c(a,ae)), xlab="Assortativity", main="")
abline(v=ae, col="red")
```

![](GitHubMD_Exercise10_solution_FollowerNetwork_files/figure-markdown_github/unnamed-chunk-6-1.png)

``` r
## Shall we clean the isolated nodes and repeat all tests?
#which(!(vertices$id %in% edges$from)&!(vertices$id %in% edges$to))
# isolated node's id
#vertices$id[which(!(vertices$id %in% edges$from)&!(vertices$id %in% edges$to))]
#usersdf[which(usersdf$id==vertices$id[which(!(vertices$id %in% edges$from)&!(vertices$id %in% edges$to))]),]
#isoUsr <- lookupUsers(vertices$id[which(!(vertices$id %in% edges$from)&!(vertices$id %in% edges$to))])
#isoUsrFollowerIDs <- isoUsr$`1882314842`$getFollowerIDs()
#sum(isoUsrFollowerIDs %in% usersdf$id)     # none of the follower retrieved for this user is among the politician candidates
fcg <- graph_from_data_frame(edges,vertices=vertices[which((vertices$id %in% edges$from) | (vertices$id %in% edges$to)),])

plot(fcg, vertex.label.color="black", vertex.label.cex=0.4, layout=layout.auto, vertex.size=5, edge.curved=0.1, edge.width=1, edge.arrow.size=0.3)
```

![](GitHubMD_Exercise10_solution_FollowerNetwork_files/figure-markdown_github/unnamed-chunk-7-1.png)

``` r
ae <- assortativity_nominal(fcg, types=as.factor(V(fcg)$party))
ae
```

    ## [1] 0.3412593

``` r
V(fcg)$screenName[degree(fcg,mode = "in")==max(degree(fcg,mode = "in"))]
```

    ## [1] "cwasi"

``` r
V(fcg)$screenName[degree(fcg,mode = "out")==max(degree(fcg,mode = "out"))]
```

    ## [1] "cwasi"    "felixzrh"

``` r
fcgCommu <- walktrap.community(fcg, steps = 50)
#print(fcgCommu)
#plot(fcgCommu, fcg, layout=layout.auto, vertex.color=fcgCommu$membership)

plot(fcg, layout=layout.auto, vertex.size=.3+.6*sqrt(graph.strength(fcg)),
     vertex.color=fcgCommu$membership, vertex.frame.color=fcgCommu$membership,
     edge.width=.2, edge.arrow.size=.2)
```

![](GitHubMD_Exercise10_solution_FollowerNetwork_files/figure-markdown_github/unnamed-chunk-7-2.png)