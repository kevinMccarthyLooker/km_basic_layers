# Using Refinements to add Period over Period functionality to existing explores

In this article, we’ll show how you can use **refinements** to implement a commonly requested feature: Period over Period analysis.

| Background on Refinements in general | Period over Period in general |
|-|-|
| With refinements, feature logic can be kept fully separate from core logic, yet can still be added to the original objects (where desired). For example, with this pattern you will be adding Period over Period capability seamlessly onto existing date fields in your existing explores, without complicating your core objects.<br><br>For additional explanation and background, see [the lookml refinements docs page](https://docs.looker.com/data-modeling/learning-lookml/refinements) and [Fabio’s Organize Lookml with Refinements](https://discourse.looker.com/t/organizing-your-lookml-into-layers-with-our-new-refinement-syntax/17489). | Period over Period is a oft-requested capability, and several other Looker methods have been described in other articles such as [ Flexible Period-over-Period Analysis](https://help.looker.com/hc/en-us/articles/360023799293--Analytic-Block-Flexible-Period-over-Period-Analysis), and [An explore filter based approach](https://help.looker.com/hc/en-us/articles/360001189687-How-to-do-Period-over-Period-Analysis), among others.  <br><br>This approach is fairly simple to build out, does not need special maintenance. For users, it flexible without being overly complex. That said, other approaches may make more sense for some use cases, e.g. where only a specific narrow set of fields is required with Period over Period.

# What Will This Period over Period Interface Look Like?
The user will pivot on a special new dimension, and will have **the option** to customize the period length and # of periods to show.
<details><summary>Expand to see two examples, including use of the optional parameters</summary>

<img src="https://discourse.looker.com/uploads/default/original/2X/e/ec6d5a9406fa26e518f3eefe2589b85f46336a47.jpeg" alt="Usage" width="800" height="600">

Here's another example where the users has chosen more periods, and created a current vs prior year comparison line graph, with a daily breakdown.  The tooltips highlight an example of the same data point being represented in multiple periods.

<img src="https://discourse.looker.com/uploads/default/original/2X/1/1998408b34d506741b54e1c15735e6dc0704f022.jpeg" alt="Advanced Usage" width="800" height="600">

</details>

# Conceptual Explanation of this approach for Period over Period

Conceptually, you'll duplicate the original dataset for each included prior period you will include. Then dates for the prior period data are offset so that it they are presented in alignment with the current period.
<img src="https://discourse.looker.com/uploads/default/original/2X/7/71c3f46abbc5443982a2b16470f7cfd7d1dbd8f3.jpeg" alt="Conceptual Daigram" width="800" height="275">

# Implementation High Level

* First, you will paste some Period over Period Support LookML in a new file.  This is generic code which you will be able to use with any number of pop enabled fields.

* Then, you will refine your view that has the base date field, by pasting the template below and then updating the relevant references (to match your view name and date field name).

* Finally, you will import the refinement into the file where your explore is defined, and enable the PoP feature.

# Additional Complexities

While the conceptual approach is exceptionally simple, there are a couple things to watch out for:

* Database syntax differences. If not using redshift, you will likely need to swap in your dialect’s version of the timestamp_add function (i.e. tweak the sql logic we use to refine your base date for PoP).
* <details><summary>Expand for additional complexities you can also read about in the code comments</summary>

  * As you can see in the example above, this process will show ‘Prior data for future periods’ which have not yet come to pass. Technically this is an accurate representation of the data but may be distracting to users. The code provides a somewhat complex mechanism to suppress those rows automatically.

  * Timezone conversion needs to happen BEFORE date manipulation in order to maintain correct groupings, so we’ll need to apply timezone conversion with liquid instead of letting looker do convert_tz:yes.

  * PoP functionality can be added on additional date fields, though it requires some care to avoid name collisions on pop pivot dimensions you create.
</details>

# Implementation At Last
### First, you will paste some Period over Period Support LookML in a new file.  This is generic code which you will be able to use with any number of pop enabled fields.
<details><summary>See Period over Period Support LookML</summary>
```

#You should not need to modify the code below.  Save this code in a file and include that file wherever needed (i.e. where your explores are defined)
view: pop_support {
  view_label: "PoP Support - Overrides and Tools"
  derived_table: {
    sql:
select periods_ago from
(
  select 0 as periods_ago
  {% if periods_ago._in_query%}{%comment%}extra backstop to prevent unnecessary fannouts if this view gets joined for any reason but periods_ago isn't actually used.{%endcomment%}
  {% for i in (1..52)%} union select {{i}}{%endfor%}{%comment%}Up to 52 weeks.  Number can be set higher, no real problem except poor selections will cause a pivot so large that rendering will get bogged down{%endcomment%}
  {%endif%}
) possible_periods
where {%condition periods_ago_to_include%}periods_ago{%endcondition%}
{% if periods_ago_to_include._is_filtered == false%}and periods_ago <=1{%endif%}{%comment%}default to only one prior period{%endcomment%}
;;
  }
  dimension: periods_ago {hidden:yes type:number}
  filter: periods_ago_to_include {
    label: "PoP Periods Ago To Include"
    description: "Apply this filter to specify which past periods to compare to. Default: 0 or 1 (meaning 1 period ago and 0 periods ago(current)).  You can also use numeric filtration like Less Than or Equal To 12, etc"
    type: number
    default_value: "0,1"
  }
  parameter: period_size {
    label: "PoP Period Size"
    description: "The defaults should work intuitively (should align with the selected dimension, i.e. the grain of the rows), but you can use this if you need to specify a different offset amount.  For example, you might want to see daily results, but compare to 52 WEEKS prior"
    type: unquoted
    allowed_value: {value:"Day"}
    allowed_value: {value:"Week"}
    allowed_value: {value:"Month"}
    allowed_value: {value:"Quarter"}
    allowed_value: {value:"Year"}
    allowed_value: {value:"Default" label:"Default Based on Selection"}
    default_value: "Default"
  }
}
```
</details>

### Then, we your refine your view that has the base date field, by pasting the template below and then updating the relevant references (to match your view name and date field name).

Paste this template into a new file, and follow the instructions in the code comments.
<details><summary>See Refinement Template</summary>

Note that, in this code block, **the lines where you need to update references to match to your existing objects are left aligned**.
```
include: "*method9_pop_support__template*" #includes some fields that are core to the PoP implementation and referenced below. Ensure you update to match the location where you put the pop_support view code
view: +order_items {#!Update to point to your view name.  That view's file must be included here, and then THIS file must be included in the explore
  # The following field sets up Default Period Lengths to compare use for each of your timeframes
  # - A SIMPLIFIED ALTERNATIVE: ... If you insted wanted to limit options and simplify UI
  # - - Optional Simplify Step 1: Hardcode THIS field's sql to a single Period Length (E.g. sql:Month;; or sql:Year;;)
  # - - Optional Simplify Step 2: Hide the pop parameters by adding the following refinement to above your explore to hide pop parameters: view: +pop_support{filter: periods_ago_to_include {hidden:yes}parameter: period_size {hidden:yes}}
  # - If implementing PoP support on multiple dates in this view, embed corresponding _is_selected checks against all supported date fields.  You should probably check for lowest grain of each field before checking for next lowest grain of each field, etc
    dimension: selected_period_size {
    hidden: yes
    sql:
    {%if pop_support.period_size._parameter_value != 'Default'%}
      {{pop_support.period_size._parameter_value}}
    {%else%}
{% if    created_date._is_selected    %}Day
{% elsif created_week._is_selected    %}Week
{% elsif created_month._is_selected   %}Month
{% elsif created_quarter._is_selected %}Quarter
{% elsif created_year._is_selected    %}Year
        {%else%}Year{%comment%}none of the anticipated dimensions were selected: using year over year as default{%endcomment%}
      {%endif%}
    {%endif%}
;;
  }
dimension_group: created {#Update your base field with this logic (no modification needed other than dimension_group_name)
    convert_tz: no #need to do timezone conversion before the date manipulation, so we can't rely on looker dimesion to apply it
    sql:
    {%comment%}Optional: Delete THIS entire line if you want to always skip timezone conversion{%endcomment%}{%assign convert_tz = false %}{% assign timezone_conversion_string_size = _query._query_timezone | size %}{%if timezone_conversion_string_size > 0 %}{%assign convert_tz = true %}{%endif%}
    {% if pop_support.periods_ago._in_query %}
      dateadd(
{{selected_period_size._sql}},
      pop_support.periods_ago,
      --BEGINORIGINALDATE{%comment%}adding this keyword to facilitate liquid extraction of the original basic date logic by another field{%endcomment%}
      {%if convert_tz %}CONVERT_TIMEZONE('UTC','{{_query._query_timezone}}',{%endif%}${EXTENDED}{%if convert_tz %}){%endif%}--ENDORIGINALDATE
      )
    {%else%}{%if convert_tz %}CONVERT_TIMEZONE('UTC','{{_query._query_timezone}}',{%endif%}${EXTENDED}{%if convert_tz %}){%endif%}
    {%endif%}
;;
  }
  # The following field is letting us extract and hold onto our base fields original sql (i.e. ${EXTENDED}) so that we can do validation, etc
  # Implementing PoP functionality on MULTIPLE date fields in the same view will require you to duplicate the field and update references for each supported date
dimension: created_date_original_sql {
    hidden: yes
    #!Point to your date_raw field in the sql
sql:{{created_raw._sql | split:'--BEGINORIGINALDATE' | last | split:'--ENDORIGINALDATE' | first}};;
  }

# This is the field that people will select(pivot) to invoke POP.  We define it here rather than pop_support so we can add the group label and use the default we configured above
dimension: created_date_periods_ago_pivot {#!Update to match your base field name
label: "{% if _field._in_query%}Pop Period (Created {{selected_period_size._sql  | strip }}){%else%} Pivot for Period Over Period{%endif%}"#this complex label makes the 'PIVOT ME' instruction clear in the field picker but doesn't display it on output
group_label: "Created Date" #!Update this group label if necessary to make it falls in your date field's group_label
    required_fields: [pop_support.periods_ago]
    order_by_field: pop_support.periods_ago #sort numerically/chronologically.
    #should not need to update
    sql:
    CASE pop_support.periods_ago
      WHEN 0 THEN ' Current'
      WHEN 1 THEN pop_support.periods_ago || ' {{selected_period_size._sql  | strip }} Prior'
      ELSE        pop_support.periods_ago || ' {{selected_period_size._sql  | strip }}s Prior'
    END
    ;;
  }
  dimension: view_name {hidden:yes sql:{{_view._name}};;}#utilized to limiting 'future'
  #Optional Validation Field. Should probably be hidden after development is completed
measure: created_date_pop_raw_date_range_for_validation {
    view_label: "PoP Support - Overrides and Tools"
label: "Created Date Pop Validation - Range of Raw Dates Included"
sql:min({{created_date_original_sql._sql}})|| ' to '||max({{created_date_original_sql._sql}}) ;;
  }
}

```
</details>

### Finally, you will include the refinement into the file where your explore is defined, and enable the PoP feature.
Here's an example of applying the order_items refinement we created in the template, but your refinement could be similarly added to any existing explore that includes the view you refined.
<details><summary>See Explore Declaration Step LookML</summary>

Note that, in this code block, **the lines where you need to update references to match to your existing objects are left aligned**.
```
include: "/method9/method9" #in your case you'll need to include your file that applies the PoP refinements
explore: pop_cross {
  label: "PoP Method 9: Align Prior Periods to an existing date field, with Refinement and Cross Join"
  view_name: order_items #Note: this could be any explore. In this example we specifically refined order_items view (which has been given view label _PoP)
  view_label: "_PoP"
  #To enable pop, paste this join to your explore, either deleting the one very long line, or updating it (in the several liquid reference fields)
  join: pop_support {
    relationship:one_to_one #we are intentionally fanning out, so this should stay one_to_one
    # This join sql does the following:
    # - First two lines ensure the pop_support cross join if and only if lynchpin pivot field (periods_ago) is selected. This extra safety that we don't fire the join if the user used PoP parameters but didn't select a pop pivot field
    # - The long inner if suppresses prior periods that would otherwise be shown for the future.
    # - - NOTE: It's not strictly required... and some dialects may not support the pattern because of the embedded window function. You can remove it and data is still correct, but you may want to add an always_filter on the Pop base field to suppress the future periods
    # - - if/when implementing PoP on multiple dates in the explore, you would ideally (though, as mentioned above: not strictly required) duplicate the long inner line of liquid based join criteria, and update references to relate to the alternative date field
    sql:
    {% if pop_support.periods_ago._in_query%}
      LEFT JOIN pop_support on 1=1
{% if order_items.created_date_periods_ago_pivot._in_query %}and {{order_items.created_raw._sql}}<=(select max(max({{order_items.created_date_original_sql._sql}})) over() from {{order_items.view_name._sql}}){%endif%}
    {%endif%}
    ;;
  }
}
```
</details>

# Closing
I hope this pattern helps your team quickly implement Period over Period functionality without complicating your base code or adding new explores just for PoP.

Additionally, I hope this inspires you to use refinements for better code organization and feature management.

Let us know how you used this pop approach or refinements in your case!
