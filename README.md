# Send telemetry logs with User Defined Functions to Azure Application Insights

This snippet demonstrates how to implement telemetry logging in Power Apps using User Defined Functions (UDFs) and Azure Application Insights. It showcases the definition and application of UDFs to efficiently manage and send telemetry data.

![preview](./assets/udf-trace.png)

## Problem statement

Telemetry collection is crucial for developing and enhancing apps. In Power Apps, developers use the `Trace()` function to send telemetry logs to Azure Application Insights.

However, using this function independently often results in inconsistencies in trace parameters, message formats, and severity levels. These variations can lead to errors and lack a standardized approach, affecting the reliability and usefulness of the data collected.

## Solution

By implementing a standardized `User Defined Function (UDF)` for telemetry, it's possible to ensure consistent data collection, minimize errors, and improve the analysis and utility of application insights.

## Prerequisites

1. Existing Azure Application Insights resource. Add Azure Application Insights **Connection String** into the Power Apps Canvas app.
![Instrumentation Key](./assets/key.png)

2. Make sure that experimental features are enabled: 
   - User-defined functions
   - User-defined types
   ![experimental-features](./assets/experimental-features.png)

2. Power Apps Canvas app with the following controls:
   - DatePicker
   - Slider
   - Gallery
   - Radio
   - Dropdown
   - Toggle
   - Rating
   - Combobox
   - Checkbox
   - Button

## Minimal path to awesome

1. Add the following code into `Formula` property on the `App` level:

```
nfToggleOn = "Reserve a parking space along with the workspace";
nfToggleOff = "Parking space not required";

tpGalleryInputTable := Type([{title: Text}]); // Replace title with your gallery property name

tpGeneralInputTable := Type([{Value: Text}]);

udfTrace(
    datePicker: Date,
    slider: Number,
    galleryItems: tpGalleryInputTable,
    radio: tpGeneralInputTable,
    dropdown: tpGeneralInputTable,
    toggle: Boolean,
    rating: Number,
    combobox: tpGeneralInputTable,
    checkbox: Text
):Boolean =
{
    With(
        {
            // Transform Gallery Items into JSON
            wthGalleryItemsJsonOutput:
            Concat(
                ForAll(
                    Sequence(CountRows(galleryItems)),
                    {
                        tempRowIndex: Value - 1,
                        tempTitle: Last(FirstN(galleryItems, Value)).title // Replace title with your gallery property name
                    }
                ),
                """" & Text(tempRowIndex) & """:""" & Substitute(tempTitle, """", "\""") & """",
                ","
            ),

            // Transform Gallery Items into String
            wthGalleryItemsString:
            Concat(galleryItems,title,", "), // Replace title with your gallery property name

            // Transform Combobox Items into JSON
            wthComboboxItemsJsonOutput:
            Concat(
                ForAll(
                    Sequence(CountRows(combobox)),
                    {
                        tempRowIndex: Value - 1,
                        tempTitle: Last(FirstN(combobox, Value)).Value
                    }
                ),
                """" & Text(tempRowIndex) & """:""" & Substitute(tempTitle, """", "\""") & """",
                ","
            ),

            // Transform Combobox Items into String
            wthComboboxItemsString:
            Concat(combobox,Value,", ")
        },
        Trace(
            "udf_trace_demo",
            TraceSeverity.Information,
            {
                datePicker: Text(DateValue(datePicker), DateTimeFormat.ShortDate),
                slider: slider,
                gallery_items_record: "{" & wthGalleryItemsJsonOutput & "}",
                gallery_items_string: wthGalleryItemsString,
                radio: Text(First(radio).Value),
                dropdown: Text(First(dropdown).Value),
                toggle: If(
                    toggle,
                    nfToggleOn,
                    nfToggleOff
                ),
                rating: rating,
                combobox_record: "{" & wthComboboxItemsJsonOutput & "}",
                combobox_string: wthComboboxItemsString,
                checkbox: checkbox  
            }
        )
    )
};

udfTraceError(
    screen: Text,
    button: Text 
):Void =
{
    Trace(
        "udf_trace_onError",
        TraceSeverity.Error,
        {
            screen: screen,
            button: button  
        }
    )
};
```

2. Insert the following code into the `OnSelect` property of the Button:

```
udfTrace(
    DatePicker.SelectedDate,
    Slider.Value,
    ShowColumns(
        Filter(
            Gallery.AllItems,
            status = "Yes" // Replace status with your gallery property name
        ),
        title // Replace title with your gallery property name
    ),
    RadioGroup.SelectedItems,
    Dropdown.SelectedItems,
    Toggle.Checked,
    Rating.Value,
    Combobox.SelectedItems,
    Checkbox.Checked  
)
```

3. Save the app and publish it.

4. Play the app, test the button to send telemetry to Azure Application Insights.

5. Navigate to the Azure Application Insights resource. Open the `Logs` under `Monitoring`. Select `trace` table. You will see the collected telemetry under the `customDimensions`.
![Azure Application Insights](./assets/aap.png)
