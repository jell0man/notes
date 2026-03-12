
```
=====================================================================
	                    REGEX CHEAT SHEET
=====================================================================

1. THE BUILDING BLOCKS (What to look for)
---------------------------------------------------------------------
 .    | Any character (except newline) | e.g., .* (skip junk)
 \d   | A digit (0-9)                  | e.g., \d+ (port number)
 \w   | A word character (a-zA-Z0-9_)  | e.g., \w+ (username)
 \s   | Whitespace (space, tab)        | e.g., \s+ (messy spacing)
 \S   | Non-whitespace                 | e.g., \S+ (URL or IP)
 \    | Escape character               | e.g., \. (actual period)


2. QUANTIFIERS & ANCHORS (How many, and where?)
---------------------------------------------------------------------
 ^    | Start of a line                | e.g., ^Admin
 $    | End of a line                  | e.g., .exe$
 +    | One or more times              | e.g., \d+ (matches 1 or 99)
 *    | Zero or more times             | e.g., \s* (optional spaces)
 ?    | Non-greedy (stop early)        | e.g., .*? (prevents over-matching)
{n,m} | Between 'n' and 'm' times      | e.g., \d{1,3} (1 to 3 digits)


3. CAPTURE GROUPS ()
---------------------------------------------------------------------
 (...) | Match the wider pattern, but ONLY extract what is inside 
       | the parentheses.


=====================================================================
                        REAL-WORLD EXAMPLE
=====================================================================
TARGET: 
Extract the 32-character NTLM hash from a Mimikatz/SAM dump line.

STRING: 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

REGEX: 
^.*?:.*?:.*?:([a-fA-F0-9]{32})

BREAKDOWN:
1. ^            -> Start at the beginning of the line
2. .*?:         -> Skip everything until the first colon (do this 3x)
3. ([...]{32})  -> Capture exactly 32 hex characters (a-f, A-F, 0-9)
=====================================================================
```