# Redshift Import Plugin (Alpha)

> **⚠️ This plugin is not ready for production use yet!**

Import data from a Redshift table in the form of PostHog events.
## Instructions 

### 1. Select a Redshift table to use for this plugin
### 2. Create a user with sufficient priviledges to read data from your table

We need to create a new table to store events and execute `INSERT` queries. You can and should block us from doing anything else on any other tables. Giving us table creation permissions should be enough to ensure this:

```sql
CREATE USER posthog WITH PASSWORD '123456yZ';
GRANT CREATE ON DATABASE your_database TO posthog;
```
### 3. Add the connection details at the plugin configuration step in PostHog

### 4. Determine what transformation to apply to your data

This plugin receives the data from your table and transforms it to create a PostHog-compatible event. To do this, you must select a transformation to apply to your data. If none of the transformations below suit your use case, feel free to contribute one via a PR to this repo.

> **Important:** Make sure your Redshift table has a [sort key](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html) and use the sort key column as the "Order by column" in the plugin config.

#### Available Transformations

##### default

The default transformation looks for the following columns in your table: `event`, `timestamp`, `distinct_id`, and `properties`, and maps them to the equivalent PostHog event fields of the same name.

**Code**

```js
async function transform (row, _) {
    const { timestamp, distinct_id, event, properties } = row
    const eventToIngest = { 
        event, 
        properties: {
            timestamp, 
            distinct_id, 
            ...JSON.parse(properties), 
            source: 'redshift_import',
        }
    }
    return eventToIngest
}
```

##### JSON Map

This transformation asks the user for a JSON file containing a map between their columns and fields of a PostHog event. For example:

```json
{
    "event_name": "event",
    "some_row": "timestamp",
    "some_other_row": "distinct_id"
}
```

**Code (Simplified\*)**

<small>*Simplified means error handling and type definitions were removed for the sake of brevity. See the full code in the [index.ts](/index.ts) file</small>

```js
async function transform (row, { attachments }) {            
    let rowToEventMap = JSON.parse(attachments.rowToEventMap.contents.toString())

    const eventToIngest = {
        event: '',
        properties: {}
    }

    for (const [colName, colValue] of Object.entries(row)) {
        if (!rowToEventMap[colName]) {
            continue
        }
        if (rowToEventMap[colName] === 'event') {
            eventToIngest.event = colValue
        } else {
            eventToIngest.properties[rowToEventMap[colName]] = colValue
        }
    }

    return eventToIngest
}
```