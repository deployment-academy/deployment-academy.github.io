# Custom Shortcode docs

## notice

|Parameter|Description|
|---------|-----------|
|type|warning, info, note, tip|
|id|used as the HTML `div#id` of the element that wraps the notice|
|title|descriptive text added to the top of the box|

Example

```markdown
{{< notice type="tip" id="tip-id" title="My title" >}}
some content
{{< /notice >}}
```
