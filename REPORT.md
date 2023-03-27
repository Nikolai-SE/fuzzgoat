# Fuzzing with AFL 
### _by Nikolai Stoiko_

4 synchronized instances of AFL were working about 30 minutes. 

```shell
afl-fuzz -i in -o out -M fuzzer01 ./fuzzgoat @@ 
afl-fuzz -i in -o out -S fuzzer03 ./fuzzgoat @@ 
afl-fuzz -i in -o out -S fuzzer04 ./fuzzgoat @@ 
afl-fuzz -i in -o out -S fuzzer04 ./fuzzgoat @@ 
```

![Test_multithread.png](content%2FTest_multithread.png)

## Fails on input 

1. `""`

    ```c
    string:
    free(): invalid pointer
    Signal: SIGABRT (Aborted)
    ```
    
    ```// fuzzgoat.c: 265 line
    /******************************************************************************
    WARNING: Fuzzgoat Vulnerability
    
    The code below decrements the pointer to the JSON string if the string
    is empty. After decrementing, the program tries to call mem_free on the
    pointer, which no longer references the JSON string.
    
    Diff       - Added: if (!value->u.string.length) value->u.string.ptr--;
    Payload    - An empty JSON string : ""
    Input File - emptyString
    Triggers   - Invalid free on decremented value->u.string.ptr
    ******************************************************************************/
    ```
   ```c
                if (!value->u.string.length){
                  value->u.string.ptr--;
                }
    ```

2. `{"":""}`

```object[0].name =
string:
Signal: SIGSEGV (Segmentation fault)
```
 fuzzgoat.c: 245 line
 Using of invalid value will fail at 222 line.
```c
/******************************************************************************
WARNING: Fuzzgoat Vulnerability

The line of code below incorrectly decrements the value of
value->u.object.length, causing an invalid read when attempting to free the
memory space in the if-statement above.

Diff       - [--value->u.object.length] --> [value->u.object.length--]
Payload    - Any valid JSON object : {"":0}
Input File - validObject
Triggers   - Invalid free in the above if-statement
******************************************************************************/

            value = value->u.object.values [value->u.object.length--].value;
/****** END vulnerable code **************************************************/
```

3. `[""]`

```c
array
string:
free(): invalid pointer
Signal: SIGABRT (Aborted)
```

Error from 1. due json_value_free_ex


3. ```
   {"":
   4}
   ```

```object[0].name =
int:          4
Signal: SIGSEGV (Segmentation fault)
```
Like error at 2.


4. `"\""`
```
string: "
Signal: SIGSEGV (Segmentation fault)
```
fuzzgoat.c: 284 line
```c
/******************************************************************************
WARNING: Fuzzgoat Vulnerability

The code below creates and dereferences a NULL pointer if the string
is of length one.

Diff       - Check for one byte string - create and dereference a NULL pointer
Payload    - A JSON string of length one : "A"
Input File - oneByteString
Triggers   - NULL pointer dereference
******************************************************************************/

            if (value->u.string.length == 1) {
              char *null_pointer = NULL;
              printf ("%d", *null_pointer);
            }
/****** END vulnerable code **************************************************/
```

5. `{"a":"b"}`
```
object[0].name = a
string: b

Process finished with exit code 139 (interrupted by signal 11: SIGSEGV)
```
Like at 2.


7. `{"%:":888.888888,"":888E88888,"":88888888888.888888,"":888888888,"":8888888888,"":88,"":888.8888888888888}`

```
object[0].name = %:
  double: 888.888888
object[1].name =
double: inf
object[2].name =
double: 88888888888.888885
object[3].name =
int:  888888888
object[4].name =
int: 8888888888
object[5].name =
int:         88
object[6].name =
double: 888.888889

Process finished with exit code 139 (interrupted by signal 11: SIGSEGV)
```
 Like at 2.

 `BTW` Why value->u.object.lenght is 10838 ??


8. `"@"`
```
string: @
Signal: SIGSEGV (Segmentation fault)
```
 Like 4.

9. `[",",27,233]`
```
array
string: ,
int:         27
int:        233
Signal: SIGSEGV (Segmentation fault)

Process finished with exit code -1
```

 Successfully free memory for 233 and 27, but fails on "," like at 5.


10. `[[7.317,""],97,22]`
```
array
array
double: 7.317000
string:
int:         97
int:         22
free(): invalid pointer
Signal: SIGABRT (Aborted)
```
Fails on free "" like at 1.


11. ```
    [
    ]
    ```


Fails due json_parse at main.c:156
With calling `json_value_free_ex( &state.settings, root ) at fuzzgoat.c:1065 after goto`; `state.settings` seems to be invalid.


## BTW

Closing of null pointer at main.c: 144 will caused segfault.
