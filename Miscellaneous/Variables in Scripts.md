
Sometimes we will run into scripts (usually bash) that will mentions if-statements for Variables that were NOT declared in the script itself...

Example script
```
if [ -z $VAR ];then
	VAR=false

if $VAR;then
	echo 'I am true!!!'
```

nowhere in the script was $VAR set equal to true so we would have to specify it

we can usually do so in the same line we run the script

Running script
```
VAR=true sudo ./script.sh
```

We can also the variable in the enviornment:
```
export VAR=<stuff>
```


#### Poor Validation
Sometimes scripts will ask the user for input and compare the input to a variable:

Example:
```
if [[ var1 == var2 ]]; then
	do thing
fi
```

If == is used, we can use wildcards and it will work

if we need to retrieve a variable, we could potentially brute force it. see [[Codify]]

