# Intro 

So your dashboard is slow? We are doing our best but maybe it's time to think if the issue isn't on your side.

Some of our dashboards looks like https://www.theworldsworstwebsiteever.com/. Are you sure you really need to see all the time all the data? 
Or do you really need to see data for last week or month all the time? How often do you use filtering by some of those variables in your dashboard?

Let's not end up as your mother saying:
> But.. but.. I might need this in future so let's leave this icon on the Desktop.

You need to think about Prometheus as just another database where queries needs to be optimized and unneeded traffic to be avoided.

# How to debug slow Grafana dashboard
This should guide you through basic steps how to optimize your dashboard which should solve most of the cases.

## Find out slow dashboards
You probably know that something is slow since you are reading this but still.
To find out slow dashboards you can use saved Kibana search **Grafana slow query log 10s+**_(grafana logs dashboard load time and user that loaded it so you can make a graph of this in elk stack or something like this)_(possibly tune the time range and threshold). This search shows you exact Grafana dashboard link and user who loaded it which can ease your debugging.

## What is slow 

### On load I see empty page not even empty graphs
If you load Grafana dashboard and you don't see any panels or graphs just the sidebar (as  shown on this picture [image](long-loading-variables-time.png)
) it means your [templating variables](http://docs.grafana.org/reference/templating/#variables) values takes long time to load. Your dashboard won't load until all the variable values are obtained.



1. Open network console in explorer.
1. Find longest running queries.
1. Open variables settings and alter the queries according to this:

- Check datasource of the variable query. It should be set to `Trickster`.
- Best way to query variable values is using `label_values(<label_name>)` which queries Prometheus endpoint `/api/v1/label/<label_name>/values` which is read from index and is cheap and really fast.
- **Try to avoid using `label_values(<query>, <label_name>)`** it sends request to Prometheus endpoint `/api/v1/series` which is not cached and can be very expensive if there is lot of series matching that query. **Use it only for metrics with small cardinality.** There might be other metrics having the required label. 
- If you need to use `label_values(<query>, <label_name>)` so you can filter out some label values than consider rather using `query_result(<query>)` and than filtering it using regex `<label_name>="([^"]+)"`. This can be potentially slightly more expensive but on the other hand we have Trickster proxy/cache which caches those data and makes it really fast.

Try above mentioned steps and benchmark variable queries using network console.
If the dashboard is still loading slow maybe it's also time to consider if **you really need all the variables?** Maybe one dashboard for daily usage with less variables is sufficient and than you can create another allowing this in depth debugging where you would expect slower loading time.

# Dashboard loads but data takes forever to appear
Here it depends mainly on the queries itself. There is not much to do to optimize things in PromQL but one way to dramatically increase loading time is using [recording rules](https://prometheus.io/docs/practices/rules/).

## Recording rules
allows you to add your query to Prometheus and instead of calculating the query for the intended time range on dashboard load Prometheus calculates this query continuously and stores the result in new timeserie. Than you can visualize just the one timeserie which should be almost instant. 

## Datasources
Always use datasource named `default` in panels (it's default value so if you don't change it it should be ok). We manage setting of the `default` datasource and it should work always best for you and avoid any issues in the future. 

## Do you need to see all the data for that long time range?
Next way to ptimize loading time is to reconsider what graphs do you have and in what time range. Grafana allows you to [override time range for each panel](http://docs.grafana.org/reference/timerange/#panel-time-overrides-timeshift) so if you want to see total requests in last week it's ok but you don't need to load all other panels for the whole week if it's not required and just override the one panel time range(see [image](/uploads/-/system/personal_snippet/402/9ce7fd3d00ac32b25a52bcdf79acb901/image.png)).

## Do you need to see all the data all the time?
Next thing you can use is adding [rows](http://docs.grafana.org/guides/basic_concepts/#row). Rows can be collapsed and should not be loaded until expanded. This allows you to have graphs prepared and easily accessible when needed but not loading all the time ([image](/uploads/-/system/personal_snippet/402/4b3400cb38c68b74b62a6fe6c632e1a1/image.png)). 

## How fresh data do you really need?
If your dashboard shows data from last 7 days you probably don't need to reload it every 30s. Not even every minute since the data resolution is so small you won't see the so recent changes. Five minutes will do it.

# Conclusion
Please bare in mind that you are not the only one using the monitoring stack and for SRE it's critical to help them with debugging. Unfortunately Prometheus does not provide possibility to limit queries so it's up to us to treat it well. Think about what queries you run and if you really need it the same way you optimize queries to your production databases.


# References
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana documentation](http://docs.grafana.org/)
- [Trickster documentation](https://github.com/Comcast/trickster/blob/master/README.md)
- [Thanos documentation](https://github.com/improbable-eng/thanos/blob/master/README.md)
