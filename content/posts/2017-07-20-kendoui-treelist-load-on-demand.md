---
title: "KendoUI TreeList Load on Demand"
date: 2017-06-20
draft: false
tags: ["javascript", "kendoui"]
summary: "How to lazily load the descendant items of a record in a TreeList."
aliases: 
    - /kendoui-treelist-load-on-demand
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2017-07-20-kendoui-treelist-load-on-demand.md"
---

The [TreeList](http://demos.telerik.com/kendo-ui/treelist/index) isn't among the most popular KendoUI widgets, yet it's very useful and functional. It displays a hierarchically structured data, where each record has an optional relation to another record. The output is a grid which renders its rows in a tree-like fashion. In this blog post we'll see how to load the descendant items of a record on demand.

## The Problem

By default the TreeList expects us to provide the complete data set on initial load. In a real-world scenario, however, the amount of records tends to get quite large thus loading everything within a single API call would be inefficient and might hinder the app's usability. A common way to tackle such case is by adapting the widget to **load the data on demand**, that is we fetch and render the descendant items of a record only when the user has intended to see them. Being familiar with other KendoUI widgets like the TreeView or the Grid one might assume that there's an obvious and straightforward way to achieve this, but that's not the case with the TreeList.


### No ```loadOnDemand``` congfiguration flag

Unlike the [TreeView](http://docs.telerik.com/kendo-ui/api/javascript/ui/treeview#configuration-loadOnDemand), the [TreeList](http://docs.telerik.com/kendo-ui/api/javascript/ui/treelist) **doesn't** provide a configuration option to indicate that we're going to lazily load the widget data. Instead we have to resort to its data source.

## TreeListDataSource

The TreeList has its own DataSource implementation, which extends the KendoUI DataSource. Unlike the TreeView which works with a hierarchical data structure, it expects a linear data structure, i.e. JavaScript array, and the relations are described using property references. So for example the following collection:

```javascript

var dataSource = new kendo.data.TreeListDataSource({
    data: [
        { employeeId: 1, name: 'Montgomery Burns', reportsTo: null, hasSubordinates: true },
        { employeeId: 2, name: "Homer Simpson", reportsTo: 1, hasSubordinates: false },
        { employeeId: 3, name: "Lenford Leonard", reportsTo: 1, hasSubordinates: false }
    ]
});

```

will result in the corresponding tree: 

```
+-- Montgomery Burns  
|   +-- Homer Simpson  
|   +-- Lenford Leonard
```

### Remote Data Binding

For remote data binding we need to use the DataSource's ```transport``` property. This is straightforward as in any other Kendo widget: 

```javascript
var treeListDataSource = new kendo.data.TreeListDataSource({
    transport: {
        read: {
            url: 'api/employees',
            type: 'GET',
            dataType: 'json'
        }
    },
    schema: { /* ... */ }
});
```

### ```schema.model```

Next we have adapt the widget to the structure of the data by configuring the schema and describe its field mapping in the model.

```javascript

var treeListDataSource = new kendo.data.TreeListDataSource({
    transport: { /* ... */ },
    schema: {
        // property mapping
        model: {
            id: 'employeeId', // unique id field name
            parentId: 'reportsTo', // relation with another record's id
            hasChildren: 'hasSubordinates', // indicates that there're records that haven't been loaded
            fields: {
            	employeeId: { type: 'number', nullable: false },
                name: { type: 'string', nullable: false },
                reportsTo: { type: 'number', nullable: true }
            }
        }
    }
});

```

### ```hasChildren```

This property indicates whether it has child items that have to be additionally fetched from the remote service. We can map it to another property, compute it dynamically with a function or hardcode it. 

```javascript

// hard-code that the item will always have children
hasChildren: true

// map the hasChildren property to the hasSubordinates field, serialized from the server
hasChildren: "hasSubordinates"

// compute whether the given item will have children
hasChildren: function(item) {
    return item.hasEmployees && item.relatedDepartment;
}

```

## The Endpoint

On initial load the widget's data soruce will make an API call to the url we've defined, e.g.

```
GET 'api/employees'
RESPONSE: [{ employeeId: 1, name: 'Montgomery Burns', reportsTo: null, hasSubordinates: true }]
```

thus retrieving and populating the widget with the top-level records, namely the ones the ```parentId``` of which is null or not defined. After that when a record is expanded and its ```hasChildren``` property is ```true`` the data source will perform an API call but this time by also providing an additional id query parameter, e.g.

```
GET 'api/employees?id=1'
RESPONSE: [
    { employeeId: 2, name: "Homer Simpson", reportsTo: 1, hasSubordinates: false },
    { employeeId: 3, name: "Lenford Leonard", reportsTo: 1, hasSubordinates: false }]
```

In this case the endpoint should return only the descendands of the record with id of 1. Once they are loaded, the DataSource caches them in memory and does not send a request for the same record. If the default url format is not suitable to the remote service, we can use the [transport.parameterMap](http://docs.telerik.com/kendo-ui/api/javascript/data/datasource#configuration-transport.parameterMap) to convert it.
You can find the complete example [here](https://jsfiddle.net/8xwhv0Le/1/).
