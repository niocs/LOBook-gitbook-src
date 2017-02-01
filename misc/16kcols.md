# Project : Increasing the limit of number of columns from 1024 to 16384 in Calc

## Motivation
Often we need the ability to load sheets from MS Excel/csv files that have more than 1024 columns in it. MS Excel can effortlessly open such large column files, but in Calc we are limited to loading files with 1024 columns.

## Basics on the related code
The very basic classes involved are :
1. `ScDocument` - which represents a document. See its declaration [here](http://opengrok.libreoffice.org/xref/core/sc/inc/document.hxx#263)
2. `ScTable` - which represents a sheet within a document, declared [here](http://opengrok.libreoffice.org/xref/core/sc/inc/table.hxx#119)
3. `ScColumn` - represents a single column within a sheet, declared [here](http://opengrok.libreoffice.org/xref/core/sc/inc/column.hxx#119).

## Challenges
Before the work on 16k cols project started, the columns of a sheet were allocated statically as a plain array of `ScColumn`'s of size size `MAXCOLCOUNT` inside `ScTable` called `aCol`. We could have just altered the definition of `MAXCOLCOUNT` from 1024 to 16384, but it has the following two major issues :

1. Since we preallocate `MAXCOLCOUNT` number of columns at the beginning, the memory requirement for a single sheet will increase a lot, independent of how much columns the user is actually going to use.
2. Most of the code in methods of `ScTable` and `ScDocument` has the following code-pattern in them :
```cpp
for ( SCCOL nCol = 0; nCol <= MAXCOL; ++nCol )
{
    // Does some work on the column aCol[nCol]
}
```
where `MAXCOL` is defined as `MAXCOLCOUNT - 1`. If we increase the `MAXCOLCOUNT` statically, it would cause these loops to take longer even though the user may use only some of the first few columns. So the change affects even the users who just want to work with documents with few columns.

The obvious way to mitigate the above problems is to make the column storage in `ScTable` dynamic, such that a column is allocated if it is actually required by the user.

## What have been done so far ?

0. A [discussion](http://nabble.documentfoundation.org/tdf-50916-Calc-Dynamic-column-container-td4162663.html) on LO dev mailing list that led to the ideas to work on this project. This is a great resource for anyone starting out to work in this project.

1. First step in this project was to introduce a dynamic column container using `std::vector` in `ScTable` instead of using a plain static array. This was done in the patch [Dynamic column container in the pursuit of tdf#50916](https://gerrit.libreoffice.org/#/c/21620/). The dynamic container is used in `ScTable` such that there is minimal change required for other files. But it is still not possible to let this container be dynamic (that is to grow from no columns to the required number of columns on user need), because the code in most places assumes that the container has `MAXCOLCOUNT` number of columns, so the dynamic container still needs to be initialized with `MAXCOLCOUNT` number of columns till we fix all places where the "static column container" assumption is made.

2. Some side steps were then made to improve some static datastructures used in `ScMarkData` and `ScAttrArray` classes to work efficiently with dynamic column container and future column count increase. These were done in two patches a) [Refactor ScMarkData for tdf#50916](https://gerrit.libreoffice.org/#/c/22163/) and b) [Refactor ScAttrArray for tdf#50916](https://gerrit.libreoffice.org/#/c/27828/). Be sure to read the detailed commit messages.

3. A first step to fix the places where "static column container" assumption is made is done in the patch [tdf#50916 : Refactor table1.cxx wherever there is column access](https://gerrit.libreoffice.org/#/c/31125/). This patch fixes some of the places in `table1.cxx` where this assumption is made.

4. These patches continues the effort to fix the code where the static column container assumption is made -
   1. [Use aCol.size() instead of MAXCOL to increase max number of column](https://gerrit.libreoffice.org/32708)
   2. [Make sure that we don't access aCol out of range](https://gerrit.libreoffice.org/33061)
   3. [Allow ScTable work on dynamic ScColContainer](https://gerrit.libreoffice.org/33296),
   4. [Introduce new column validation function](https://gerrit.libreoffice.org/33196) (in review),
   5. [Allow proper updating, deleting and inserting tabs](https://gerrit.libreoffice.org/#/c/33647/) (in review)
   6. [Allow dynamically increase number of columns according to needs](https://gerrit.libreoffice.org/33724) (in review)

## What to do next ?

We need to make sure we have fixed all places in table\*.cxx and documen\*.cxx where the static assumption is made. Another type of change is where there is modification of a column (search for non-const functions in ScTable/ScDocument) - if the required column does not exist, need to create it using `ScTable::CreateColumnIfNotExists()` introduced in the currently in review patch [Allow dynamically increase number of columns according to needs](https://gerrit.libreoffice.org/33724).
