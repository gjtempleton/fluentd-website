# Parse Postfix Maillogs and Store Them in MongoDB

It is helpful to have maillogs stored in a semi-structured manner for future auditing and root cause analysis. This brief solution guide shows you how to parse Postfix maillogs and store them into MongoDB in near real-time.

## Prerequisites

- A basic understanding of Fluentd
- A Postfix MTA running locally

In this guide, we assume we are running [td-agent](/download) on Ubuntu Precise.

## Tailing the Postfix Maillog

The first step is to set up [the tail input](http://docs.fluentd.org/articles/in_tail) to tail the maillog.

The Postfix maillog looks like this:

```
14-03-26T19:49:56+09:00 worker001 postfix/smtp[13747]: 31C5C1C000C: to=<foo@example.com>,
relay=mx.example.com[127.0.0.1]:25, delay=0.74, delays=0.06/0.01/0.25/0.42, dsn=2.0.0, status=sent (250 ok dirdel)
```

Which can be parsed with the following regular expression:

```
/^(?<date>[^ ]+) (?<host>[^ ]+) (?<process>[^:]+): (?<message>((?<key>[^ :]+)[ :])? ?((to|from)=<(?<address>[^>]+)>)?.*)$/
```

Thus, assuming the maillog is located at `/var/log/maillog`, to tail and parse the maillog, add the following to the Fluentd configuration (which, for `td-agent` is at `/etc/td-agent/td-agent.conf`).

```
<source>
  type tail
  path /var/log/maillog
  tag maillog.hostname_1
  format /^(?<date>[^ ]+) (?<host>[^ ]+) (?<process>[^:]+): (?<message>((?<key>[^ :]+)[ :])? ?((to|from)=<(?<address>[^>]+)>)?.*)$/
</source>
```

(If one is interested in using a dynamic hostname, [fluent-mixin-config-placeholder](https://github.com/tagomoris/fluent-mixin-config-placeholders) can be used.)

## Outputting to MongoDB

The output plugin for MongoDB is bundled for `td-agent`. If you are running a vanilla Fluentd instance, run `gem install fluent-plugin-mongo` first.

Add the following lines to output data into MongoDB:

```
<match maillog.*>
  type copy
  <store>
    # for debug (see /var/log/td-agent.log)
    type stdout
  </store>
  <store>
    type mongo
    database fluentd #DB name
    tag_mapped true
    host YOUR_MONGODB_HOST
    port YOUR_MONGODB_PORT
    # ssl true
    # user USER
    # password PASSWORD
  </store>
</match>
```

The `tag_mapped` parameter allows Fluentd to create a collection per tag. For example, if an event comes with tag = maillog.host1, it creates a collection named `maillog.host1`, and if another event comes with tag = maillog.host2, it creates a collection named `maillog.host2`. If you want to collect all maillogs into a single collection, use the following configuration instead.

```
<match maillog.*>
  type copy
  <store>
    # for debug (see /var/log/td-agent.log)
    type stdout
  </store>
  <store>
    type mongo
    database fluentd #DB name
    collection maillog #Collection name
    host YOUR_MONGODB_HOST
    port YOUR_MONGODB_PORT
    flush_interval 10s # for testing
    # ssl true
    # user USER
    # password PASSWORD
  </store>
</match>
```

Also, notice that the MongoDB output supports SSL and password authentication (commented out).

## Restart and Confirm That Data Flow into MongoDB

Restart td-agent with `sudo service td-agent restart`. Then, run `tail` against /var/log/td-agent.log. You should see the following lines:

```
2012-07-12 15:19:03 +0000 maillog.hostname1: {"address":"foo@example.com", "date":"2012-03-26T19:49:56+09:00", "host":"worker001", "key":"31C5C1C000C", "message":"31C5C1C000C: to=<foo@example.com>, relay=mx.example.com[127.0.0.1]:25, delay=0.74, delays=0.06/0.01/0.25/0.42, dsn=2.0.0, status=sent (250 ok dirdel)", "process":"postfix/smtp[13747]"}
```

Now, go into MongoDB shell and check that the data is indeed imported.

## What's Next?

In production, you might want to remove writing output into stdout. So, use the following output configuration:

```
<match haproxy.*>
  type mongo
  database fluentd #DB name
  collection maillog #Collection name
  host YOUR_MONGODB_HOST
  port YOUR_MONGODB_PORT
  # ssl true
  # user USER
  # password PASSWORD
  </match>
```

Do you wish to store maillogs into other systems? Check out other [data outputs!](/dataoutputs).