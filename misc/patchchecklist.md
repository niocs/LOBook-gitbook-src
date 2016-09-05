# Checklist before submitting the patch

1. Check for indentation issues, always use four *space* indentation.

2. Do not extra indent the block curly braces.

3. Do not use C style casting like (ScColumn *)(pCol), use either static_cast or dynamic_cast as appropriate.

4. Do not use low level C/C++ char functions like isalpha() or isdigit() on chars from OUString as they are not
   unicode aware. See CharClass::isAlphaNumeric(), OUString class and others from docs.libreoffice.org

5. Make sure the variables are declared only in the smallest possible scope.

6. Always put a space between keywords like for, while, if, return, switch and the following parathesis. For example `for ( i = 0; i < 10; i++ )` rather than
   `for( i=0; i < 10; i++ )`.

7. Do not remove blank lines in the source code for a patch that is doing something else. Remember if your patch does X then it should only do X.
   However within the new code you add, you can add blank lines as appropriate.

8. Your patch on local machine sits in a dedicated git branch.

9. Make sure your dedicated branch has only one commit from you. If you want to modify then always use `git commit --amend <file1> <file2> ...` rather than
   creating a new commit.

10. Always review the patch before you submit to gerrit by doing "git show" after you commit everything.

11. Ensure that commit message contains the corresponding tdf bug id like `tdf#23452`

12. Do not change the Change-Id line in the commit message while doing `git commit --amend`.

