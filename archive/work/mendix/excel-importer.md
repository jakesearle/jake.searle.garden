# Importing Existing Database Values into Mendix

1. This link is a really good starting point:
<https://docs.mendix.com/howto/integration/importing-excel-documents#creating>. I used sections 3-6 in order to get it to work, but just importing the Excel sheet to get a template (Section 7) seems interesting.
3. In order to get the current values in the database, open Microsoft SQL Server Management Studio, and then navigate to the table that you want, and right-click > Select Top 1000 Rows. (You may have to change the query if the table has more than 1000 entries). Then over the results right click > "Select All", then right-click > "Copy with Headers" (Or `ctrl+A` then `ctrl+shift+C`). Then you can paste into a fresh Excel file. Make a new sheet for each table that you need.

- I found that if the table has too many entries (I think 1000+) the copy-paste results become kind of garbled. In order to get around this, right-click the results > "Save Results As..." and I just saved it as a .csv, and for some reason when you open this file in Excel, you can copy and paste the cell contents just fine into your master Excel sheet. (I'm sure there's some way to just have Mendix pull from the .csv, but I just copied the .csv into the master Excel)

4. At this point, there might be a lot of "NULL" and "0" entries in the Excel file, and these (unfortunately) are going to be processed in Mendix as Strings, which is probably not the best. Although I don't know exactly how Mendix deals with null fields, you'll need to save these as blank strings. Although you could get rid of these during a Mendix "Parse with" microflow", I just used the find and replace functionality in Excel to fix these. Creating a microflow similar to the one in section 4 [here](https://docs.mendix.com/howto/integration/importing-excel-documents#creating) would probably be best in the long run.
   - Note: After I did this, I discovered [this page](https://docs.mendix.com/refguide/special-checks) which might offer a more elegant solution by using the `empty` keyword
5. After these steps have been completed, you can continue on with the [tutorial](https://docs.mendix.com/howto/integration/importing-excel-documents#creating) that I mentioned in step 1.

6. One last note: My domain model looked like this

So essentially, I just had one object, with 5 different many to one relations (ignore the XLSFile). In my Excel sheet, I had just plain strings for the Model, Manufacturer, etc. But by following [the example on step 6.16](https://docs.mendix.com/howto/integration/importing-excel-documents#creating), Mendix just filled in these side tables as well without any hiccups. I imagine that if you had foreign keys / indices in that column, this would function differently.
7. Once you are ready to move to a free node, or a live node, you'll just need to export the template which you've made (on the Excel Import Overview page running locally), and then you'll be able to import that back into the live node on the same page. Just remember to refresh the MxModelReflection for the "System" module (and whichever module you made) *before* you import the template.

### Additional Tips

- But what if the existing table has "Created" and "Modified" columns? Can I just ignore these because Mendix has the option to have "createdDate" and "changedDate" built in?
  - Well, this is a little complicated, because when you import the table entries from Excel, Mendix will modify "createdDate" and "changedDate" so that they match the time when you imported them. Additionally, these attributes are locked by Mendix to be read-only (even for administrators, as far as I can tell), so you can't just change the template to have these "Created" and "Modified" columns be imported onto the "createdDate" and "changedDate". Generally, if the user doesn't need to see when objects were created or modified, I would think you would be fine just using the default Mendix "createdDate" and "modifiedDate", and just having them be reset when those objects are imported. However, in the project that I was building, having the user be able to see the existing "Created" and "Modified" attributes was important, so I had to create the following:
  - For the custom "Created" attribute, I created a microflow that runs after creation of an item(ex: `ACr_MyItem`), which changes the given item's `Created` attribute using a "Change item" activity with this expression: `if $MyItem/Created = empty then $MyItem/createdDate else $MyItem/Created`. This makes it so that items imported from Excel retain their created dates, but items which are created using the client have their correct "Created" value as well. (You don't have to commit or refresh these changes)
  - For the custom "Modified" attribute, I created another microflow which runs before each object's commit. (ex: `BCo_MyItem`). In this, I have a single "Change item" activity, which modifies our item's `Modified` attribute. There's no need to commit or refresh these changes, since they will be committed and refreshed anyway (hence the "before commit"). The expression looks something like this:

    ```
    if secondsBetween($MyItem/changedDate, $MyItem/createdDate) < 3
      and $MyItem/Created != $MyItem/createdDate
    then $MyItem/Modified
    else [%CurrentDateTime%]
    ```

    - Note here the use of the `secondsBetween` function. I think that when Mendix creates the object, it also takes a little amount of time to import each of the fields from Excel, so you can't just directly compare the createdDate and changedDate, as they will be *milliseconds* away from being the same. When I did directly compare them, about 1/3 of the objects were imported correctly, with the rest saying that they were last modified today.

- What if an object in my Domain Model has an AutoIncrement property? That's read only as well.
  - What I did here is similar to the solution about with the conflicts with Mendix's `createdDate` and `changedDate`. Just create an Integer type attribute in the model called something like `_OldID` (I'm aware that it's not in the screenshot of my domain model shown earlier), then have the Excel Importer list this old ID as the key, and Mendix will just use the AutoIncrement data type on your data which you import.
