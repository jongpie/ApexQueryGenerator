# Nebula Query & Search for Salesforce Apex
[![Travis CI](https://img.shields.io/travis/jongpie/NebulaQueryAndSearch/master.svg)](https://travis-ci.org/jongpie/NebulaQueryAndSearch)

<a href="https://githubsfdeploy.herokuapp.com" target="_blank">
    <img alt="Deploy to Salesforce" src="https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/deploy.png">
</a>

A dynamic SOQL query & SOSL search library for for Salesforce Apex<br /><br />

## Features
* Provides chainable builder methods for dyanmically building queries & searches in APex
* Easily add fields to a query based on field level security
* Easily add fields from a field set
* Automatically adds the parent name field for any lookup/master-detail fields
* Adds translations for picklist fields & record types by calling includeLabels()
* Adds localized formatting for number, date, datetime, time, or currency fields by calling includeFormattedValues()
* Leverage query scope to filter results
* Enable query & search caching by simple calling cacheResults()
* Reuse your dynamic SOQL queries to quickly build dynamic SOSL searches

## Overview
There are 3 main builder classes

 &nbsp; | SObjectQueryBuilder | AggregateQueryBuilder | SearchBuilder
------- | --------------------|-----------------------|--------------
Super Class | Soql.cls (Queries) | Soql.cls (Queries) | Sosl.cls (Searches) | -
Action | Queries an SObject | Queries an SObject | Searches 1 or more SObjects
Returns | `SObject` or `List<SObject>` | `AggregateResult` or `List<AggregateResult>` | `SObject`, `List<SObject>` or `List<List<SObject>>`

## SOQL SObject Query Examples
**Basic Usage:** Query an object & return the object's ID and display name field (typically the 'Name' field, but some objects use other fields, like Task.Subject and Case.CaseNumber). Since no filters have been added, this query would also return all accounts.

```
List<Account> accounts = new SObjectQueryBuilder(Schema.Account.SObjectType).getResults();
```

**Advanced Usage:** Query an object & leverage the query builder methods. The order of the builder methods does not matter - you can arrange the calls to these methods in any order that you prefer.

```
SObjectQueryBuilder accountQuery = new SObjectQueryBuilder(Schema.Account.SObjectType) // Query the account object
    .addField(Schema.Account.ParentId)                                                 // Include the ParentId field, using SObjectField. The current user must have at least read access to the field
    .addField(Schema.Account.Type, Soql.FieldCategory.UPDATEABLE)                      // Include the Type field if the current user has access to update it
    .addFields(Soql.FieldCategory.CUSTOM)                                              // Include all custom fields - only fields that are accessible to the user are included
    .addFieldSet(Schema.Account.MyFieldSet)                                            // Include all fields in a field set that are accessible to the user
    .removeField(Schema.Account.My_Custom_Field__c)                                    // remove a custom field
    .usingScope(Soql.Scope.MINE)                                                       // Set the query scope
    .filterWhere(Schema.Account.CreatedDate, '=', new Soql.DateLiteral('LAST_WEEK'))   // Filter on the created date, using a date literal
    .orderBy(Schema.Account.Type)                                                      // Order by a field API name - sort order/nulls defaults to 'Type ASC NULLS FIRST'
    .orderBy(Account.Name, Soql.SortOrder.ASCENDING)                                   // Order by, using SObjectField & sort order
    .orderBy(Account.AnnualRevenue, Soql.SortOrder.DESCENDING, false)                  // Order by, using SObjectField, sort order and nulls sort order
    .limitTo(100)                                                                      // Limit the results to 100 records
    .includeLabels()                                                                   // Include labels/translations for any picklist fields or record types. These are aliased using the convention 'FieldName__c_Label'
    .includeFormattedValues()                                                          // Include formatted values for any number, date, time, or currency fields
    .cacheResults()                                                                    // When enabled, the query results are internally cached - any subsequent calls for getResults() will returned cached results instead of executing the query again
    .offsetBy(25);                                                                     // Skip the first 25 results

// Execute the query and store the results in the 'accounts' variable
List<Account> accounts = accountQuery.getResults();

/****** Resulting output *******
SELECT Id, MyCustomDateField__c, MyCustomPicklistField__c, Name,
    format(MyCustomDateField__c) MyCustomDateField__c__Formatted,
    toLabel(MyCustomPicklistField__c) MyCustomPicklistField__c__Label
FROM Account
USING SCOPE MINE
WHERE CreatedDate = LAST_WEEK
ORDER BY Type ASC NULLS FIRST, Name ASC NULLS FIRST, AnnualRevenue DESC NULLS LAST LIMIT 100 OFFSET 25
*******************************/

System.debug(accountQuery.getQuery());
```

## SOSL Search Examples
**Basic Usage:** Search a single object

```
SObjectQueryBuilder userQuery = new SObjectQueryBuilder(Schema.User.SObjectType); // Create an instance of SObjectQueryBuilder for an SObject - you can include additional fields, filters, etc
SearchBuilder userSearch      = new SearchBuilder('my search term', userQuery);   // Create a new SearchBuilder instance with a search term & instance of SObjectQueryBuilder
List<User> userSearchResults  = userSearch.getFirstResults();                     // SearchBuilder returns a list of lists of sobjects - getFirstResults() returns the first list

/****** Resulting output *******
FIND 'my search term' IN ALL FIELDS RETURNING User(Id, Name)
*******************************/

System.debug(userSearch.getSearch());
```

**Advanced Usage:** Search several objects

```
SObjectQueryBuilder accountQuery  = new SObjectQueryBuilder(Schema.Account.SObjectType);                  // Create an instance of SObjectQueryBuilder for the Account object
SObjectQueryBuilder contactQuery  = new SObjectQueryBuilder(Schema.Contact.SObjectType);                  // Create an instance of SObjectQueryBuilder for the Contact object
SObjectQueryBuilder leadQuery     = new SObjectQueryBuilder(Schema.Lead.SObjectType);                     // Create an instance of SObjectQueryBuilder for the Lead object
List<SObjectQueryBuilder> queries = new List<SObjectQueryBuilder>{contactQuery, accountQuery, leadQuery}; // Add the SObjectQueryBuilder queries to a list

SearchBuilder mySearch            = new SearchBuilder('my search term', queries); // Create a new SearchBuilder instance with a search term & the list of SObjectQueryBuilder queries
List<List<SObject>> searchResults = mySearch.getResults();                        // Returns all search results

/****** Resulting output *******
FIND 'my search term' IN ALL FIELDS RETURNING Account(Id, Name), Contact(Id, Name), Lead(Id, Name)
*******************************/

System.debug(mySearch.getSearch());
```