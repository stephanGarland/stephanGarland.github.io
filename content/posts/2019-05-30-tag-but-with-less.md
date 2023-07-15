---
title: 'Tag, but with less running'
date: Thu, 30 May 2019 18:00:12 +0000
draft: false
tags: ['gps', 'python']
---

The last project I had during my time as a Distribution Engineer was to map out a nearby town's electrical system. They had contracted us to handle their maintenance and construction, and their maps were... well, they weren't. As an aside, here's a brief explanation of rural electric membership cooperatives (REMCs), municipal power districts (munis), and investor-owned utilities (IOUs) that nobody asked for. Feel free to skip a paragraph or three.

In the beginning, there was electricity, and it existed solely in cities. The cost to run lines out to sparsely-populated rural areas, or even smaller towns, was deemed too prohibitive for utilities trying to turn a profit. Around the late 1800s, municipal power districts began bringing public power to the people, and much to the chagrin of investor-owned utilities. Rates were set to a level that made the utility sustainable, and they worked.

In 1935, FDR signed an executive order to create the Rural Electrification Administration, which was made law by Congress in 1936. Traveling teams of electricians roamed the country, setting poles, stringing wire, and providing rural homes with electricity. In approximately 30 years, American went from ~3% of rural homes having electricity, to ~90%.

The REA did create some tension, though, in that the area around munis was now the territory of REMCs. Often, the REMC would like very much to absorb the muni's territory, but this is usually met with hostility. Sometimes, muni rates are cheaper than REMCs, so that's an obvious deterrent.

In any case, back in the present-ish day, a local muni had contracted my REMC to provide maintenance and installation for them. Their maps were on paper, which isn't unusual, they had no GIS system, and their records of which transformers fed which houses, and from what phase, were sparse. The only thing to do in a case like this is start from scratch.

One of the key pieces of data needed for a GIS system are coordinates. As I discovered, there is a dearth of software, commercial or FOSS, which provides this capability. There are handheld GPS devices which can tag locations, but there is/was nothing I could find that could output information to a CSV or similar format, which I needed to easily import the information into our database. Python to the rescue.

[gps-point-tagger](https://github.com/stephanGarland/gps-point-tagger) was the result of a few days of coding, after I learned that we had a laptop with a built-in GPS receiver. [pyserial](https://github.com/pyserial/pyserial) allows the program to access the receiver, and [pynmea2](https://github.com/Knio/pynmea2) allows the [NMEA 0183](https://en.wikipedia.org/wiki/NMEA_0183) messages from the GPS receiver to be parsed. Finally, [simplekml](https://pypi.org/project/simplekml/) allows a secondary program to map the poles to a KML file to either double-check the CSV, or quickly visualize your progress with Google Earth or a similar program. The code isn't great, but it's functional, decently commented, and works. More importantly, it is also easy to compile with py2exe or the like, which was a key design requirement I had set - I knew I was leaving, and I didn't want to rely on someone being able to figure out getting a Python environment installed. Similarly, I wanted to make sure the documentation was never lost, so I wrote up a short help file in HTML, encoded it into Base64, put it in the source, and used Python's webbrowser to call the decoded file. There are probably more elegant ways of managing it, but it worked for me.

The program itself is reasonably intuitive for anyone in the utility industry. When launched, it checks for a GPS receiver, but still allows you to launch if one isn't found - useful for development, since my machine didn't include one. You then tell it where you want to save the CSV file, and you're off. The secondary program is launched from the main, and does require simplekml to be installed.

![](/images/2019-05-30-tag-but-with-less/1.png)

The program asks for a variety of inputs, all of them optional. Not shown on this screenshot is a Get Lat/Long button, as it's only displayed if you have a receiver. Basic operation is input the pole number, stand next to it, get the current coordinates, have your lineman helper read out information about the transformer to you (or do video conferencing with two tablets, one on a pole held up to the transformer), then go to all the meters being fed from the transformer to get their information. The secondary program, called with subprocess, is a smaller version of this for mapping all of the meter locations. The checkbox allows for the pole number to be auto-incremented when the form is cleared, to save repetitive input. Similarly, Vpri, Vsec, and Overhead/Padmount fields are persistent, as they rarely change.

One tricky bit, which I've just now fixed, was getting the auto-increment to correctly work. Pole naming schemes vary wildly from utility to utility, but ours would be something like BRW4-N9, with the first two letters indicating the substation feeding the line, N/E/S/W indicating the cardinal direction of the line, and the digit indicating the pole number, ascending out from the substation. Each hyphenated suffix was a takeoff line, i.e. a branch. In this instance, the pole was the 9th in a North branch, terminating at the 4th pole on the West feed from the Brookston substation. This can be hyphenated as much as needed, e.g. BRW4-N9-W5-S1, but in practice we rarely got past two takeoffs.

The naïve approach to incrementing is to cast the last character of the string to an integer, increment it by one, and cast it back to a string to be joined. This, of course, breaks when you pass 99, since 'BRW4-N9'\[0:-1\] + str(int('BRW4-N9'\[-1:\] + 1) becomes BRW4-N910, rather than BRW4-N100. My solution was this:

```
if (entry[0] == 'gs_equipment_location' and output_check_var.get() == 1):
    text = entry[1].get()
    mod_text = str(text)
    last_ele = mod_text.split('-')[-1:][0]
    last_alpha = ''.join(filter(lambda x: not x.isdigit(), mod_text))

    if not any(x.isdigit() for x in last_ele):
        # If the last element is entirely non-numeric, don't increment
        pass
    else:
        # Given an input like "BRW4-N6", it is split into ['BRW4', 'N6']
        # The last element of the list is then converted into a string via slicing
        # A filter is run to generate the numeric and non-numeric portions with .isdigit()
        # It's all joined with map, with the original mod_text being sliced to exclude
        # the last element, last_char, and last_num incremented by one
        # Note that if the last element ends in a letter, it will be flipped, i.e.
        # "BRW4-6N" becomes "BRW4-N7"
        last_num = int(''.join(filter(lambda x: x.isdigit(), last_ele)))
        last_char = ''.join(filter(lambda x: not x.isdigit(), last_ele))

        if "-" in mod_text:
            mod_text = '-'.join(map(str, mod_text.split('-')[:-1])) + '-' + last_char + str(last_num + 1)
        # If the entry has no "-", e.g. BRW4, use this instead to avoid -BRW5
        else:
            mod_text = last_alpha + str(last_num + 1)
```

That last comment, about the last element flipping, was something that I just dealt with at the time due to a time crunch, but now, I can fix it.

Pretty simple if statement.

```
if last_ele[1].isdigit() and last_ele[0].isalpha():
    mod_text = '-'.join(map(str, mod_text.split('-')[:-1])) + '-' + last_char + str(last_num + 1)
else:
    mod_text = '-'.join(map(str, mod_text.split('-')[:-1])) + '-' + str(last_num + 1) + last_char
```

Why not improve the rest of the program, while I'm here? It handled incrementing non-alphanumeric characters in the pole name by ignoring them, and it didn't have any type checking for the rest of the fields, some of the functions are huge...

For starters, the Save and Clear/Next Entry are confusing. It's not immediately clear that Clear/Next Entry also saves the file, but even once you know that, why does Save exist? Let's combine them, and rename Clear/Next Entry to Save/Next Entry.

![](/images/2019-05-30-tag-but-with-less/1.png)

Better. Now, let's clean up clear\_entries() and wrangle\_data(). I'm not sure why I named it wrangle\_data, but here we are.

Let's make a new function for incrementing the pole number, and call it from clear\_entries(). We'll move all of the code that does the incrementing into it, and have it return the modified text field.

```
if (entry[0] == 'gs_equipment_location' and output_check_var.get() == 1):
    incremented_pole_field = increment_pole_number(entry)
    if not incremented_pole_field: # Catches increment_pole_number's none return if a typo is made
        return
```

wrangle\_data() just needs the writer.writerow(\[\]) line added, so entries have a blank line between them.

Now, how about some error checking? While I've never seen pole numbers with non-alphanumeric entries, it's possible they exist somewhere, so let's check, but allow the user to proceed with their horrifying naming scheme if they so choose. deliberate\_typo is a global initialized to None, which is called at the beginning of the function (not shown here).

```
if not any(x.isdigit() for x in last_ele):
    # If the last element is entirely non-numeric, don't increment
    pass
# Thanks to Brian on SO - https://stackoverflow.com/a/266162/4221094
elif not last_ele.translate(str.maketrans('', '', string.punctuation)) == last_ele:
    # If there are punctuation marks (assuming a typo), don't try to parse it
    # but also inquire if it was deliberate, so they can fix the error
    if not deliberate_typo:
        typo_in_pole_name = tkMsg.askyesno("Error", "Did you mean to include a non-alphanumeric character?")
        if typo_in_pole_name:
            tkMsg.showinfo("", "OK! I won't ask you again.")
            deliberate_typo = True
            pass
        else:
            deliberate_typo = False
            return
```

Also, we should check that the voltages are sane. Typical primary voltages for distribution in the US (phase-phase) are 4800, 12470, 22900, and 34500 VAC. Also, some places (like my utility) reference line voltage as phase-ground, not phase-phase, so that set would be (roughly) 2770, 7200, 13220, and 19920 VAC.\[1\] Since I'm not checking for which system they're using, let's just chuck those into a list and check that the voltage they input is in the list. Since I can, might as well throw in sub-transmission and transmission voltages into the list.

"Since I can..." Thus began a lengthy series of, "Ooh, I should catch this," and concomitant function creep, resulting in breaking them apart further, adding more try/except blocks, and so on. I eventually opted to keep the error checking to the minimum above (well, I did throw in a check that transformer kVA was an integer), keeping in mind who my customer was: experienced professionals in a specific field. If someone wants to use this tool to input their actual measured voltage, rather than rated, they probably have a good reason to do so, and I should let them. Gently ask once, but after that, hands off.

I hope you enjoyed this rambling post, and I sincerely hope that the software is of some benefit to someone else. In case you missed it at the beginning, [the repo is here](https://github.com/stephanGarland/gps-point-tagger). I welcome comments and forks alike.

\[1\] Why are these different? Three-phase power is generating three separate AC waveforms, each oscillating around a zero crossing at 60 Hz (in America). Those three sine waves are 120° apart. There's some additional math here that I'm skipping, but|tan(120° )| = sqrt(3) ~= 1.732. Divide phase-phase voltage by 1.732, and you get phase-ground voltage. This all assumes a wye transformer, which to be fair is common for distribution. For delta, phase-phase voltage = phase-ground voltage.
