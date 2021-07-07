## Background

On 2021-06-14 and 2021-06-15, a large number of users went through IDV on login.gov
but without a Service Provider. This is anomalous behavior and likely indicates
some sort of fraud.

These queries are an investigation to see if we can find any patterns

**Time limits**
Start: 2021-06-14 00:00:00
Finish: 2021-06-15 23:59:59


### First Query

Find users who started IDV, but didn't have an SP

```cloudwatch
fields
  @timestamp,
  name,
  properties.service_provider,
  properties.user_id,
  properties.user_ip,
  visit_id,
  visitor_id,
  @message
| filter name = 'IdV: intro visited'
| filter isblank(properties.service_provider)
```

Saved to: `query-1-results.csv` (not committed)

```ruby
csv = CSV.read('query-1-results.csv', headers: true)
> csv.count # number of rows
=> 220
> csv.map { |r| r['properties.user_id'] }.uniq.count # unique users
=> 179
> csv.map { |r| r['visitor_id'] }.uniq.count # unique visitors
=> 185
csv.map { |r| r['properties.user_ip'] }.uniq.count # basically 1 IP per user
=> 186
csv.map { |r| r['properties.user_id'] }.uniq # get user_ids
```

### Second query

Find users who without an SP who went to the last IDV step (who passed IDV)

```cloudwatch
fields
  @timestamp,
  properties.event_properties.success,
  name,
  properties.service_provider,
  properties.user_id,
  properties.user_ip,
  visit_id,
  visitor_id,
  @message
| filter name = 'IdV: final resolution'
| filter isblank(properties.service_provider)
| filter properties.user_id in [...] # user_ids from first query
```

Saved to: `query-2-results.csv` (not committed)

```ruby
csv2 = CSV.read('query-2-results.csv', headers: true)
> csv2.count
=> 133
> did_not_pass = csv.map { |r| r['properties.user_id'] }.uniq - csv2.map { |r| r['properties.user_id'] }
did_not_pass.count
=> 71
```

### Third query

Look up users who did not pass, and see if they had any events from an SP that day.

```cloudwatch
fields @timestamp, properties.event_properties.success, name, properties.service_provider, properties.user_id, properties.user_ip, visit_id, visitor_id, @message
| filter properties.user_id in [...did_not_pass] # user_ids
| display properties.user_id, ispresent(properties.service_provider) as has_sp, @timestamp
| stats sum(has_sp) as sum_has_sp by properties.user_id
| sort sum_has_sp asc
```

Results
- 27 never had events from an SP that day
- remaining 44 had an SP for at least one event that day

Re-running there query with user_ids from #1
- 186 users who started proofing with no SP total that day
- Out of that, 78 never had an SP that day, meaning 108 did have an SP at some other point in the day
