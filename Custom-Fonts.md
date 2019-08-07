## Breakdown
There are two main components here:

1. How to draw a character with the ```write()``` function in HyperDisplay
1. How to read the MicroView bitmap in '8x16font.h'

### Character Drawing in HyperDisplay
Fundamentally HyperDisplay only needs to know how to set a given (X,Y) pixel to the color of your choice in order to make use of all the more advanced drawing functions, including fonts. For example```line()``` calls ```pixel()``` several times to create the line. Similarly the function to draw a character ```write()``` will also call ```pixel()``` several times. Since ```pixel()``` can draw anywhere on the screen there is no such limit as what you describe when you suggest 

> the display does not support fonts with heights larger than eight pixels

To better understand how ```write()``` works let's look at the code:

```
        // FYI this virtual function can be overwritten. It is just the most basic default version
	size_t hyperdisplay::write(uint8_t val)
	{
		size_t numWritten = 0;
		getCharInfo(val, &hyperdisplayDefaultCharacter);

		// Check to see if current cursor coordinates work for the requested character
		if(((pCurrentWindow->xMax - pCurrentWindow->xMin) - hyperdisplayDefaultCharacter.xDim) < pCurrentWindow->cursorX)
		{
			if(((pCurrentWindow->yMax - pCurrentWindow->yMin) - hyperdisplayDefaultCharacter.yDim) < pCurrentWindow->cursorY)
			{
				return numWritten;	// return because there is no more room in the x or y directions of the window
			}
			pCurrentWindow->cursorX = pCurrentWindow->xReset;				// Put x cursor back to reset location
			pCurrentWindow->cursorY += hyperdisplayDefaultCharacter.yDim;	// Move the cursor down by the size of the character
		}

		// Now write the character
		if(hyperdisplayDefaultCharacter.show)
		{
			//fillFromArray(pCurrentWindow->cursorX, pCurrentWindow->cursorY, pCurrentWindow->cursorX+hyperdisplayDefaultCharacter.xDim, pCurrentWindow->cursorY+hyperdisplayDefaultCharacter.yDim, hyperdisplayDefaultCharacter.numPixels, hyperdisplayDefaultCharacter.data);
			for(uint32_t indi = 0; indi < hyperdisplayDefaultCharacter.numPixels; indi++)
			{
				pixel(((pCurrentWindow->cursorX)+*(hyperdisplayDefaultCharacter.xLoc + indi)), ((pCurrentWindow->cursorY)+*(hyperdisplayDefaultCharacter.yLoc + indi)), NULL, 1, 0);
			}

			numWritten = 1;

			// Now advance the cursor in the x direction so that you don't overwrite the work you just did
			pCurrentWindow->cursorX += hyperdisplayDefaultCharacter.xDim + 1;
		}

		pCurrentWindow->lastCharacter = hyperdisplayDefaultCharacter;	// Set this character as the previous character - the info will persist because this is direct 
		return numWritten;				
	}
```

1. Call ```getCharInfo()``` to populate the ```hyperdisplayDefaultCharacter``` structure with information for the given character ```val```
1. Check if there is room for the character to be displayed at the current cursor location, which is unique to the current window. Uses the character's ```yDim``` and ```xDim``` information.
1. Draw the character by looping through all the coordinates and drawing a pixel (using HyperDisplay's ```pixel()``` function) that takes on the window's current color setting. This uses the character's ```xLoc```, ```yLoc```, and ```numPixels``` information.

To get a different character to show up the information inside ```hyperdisplayDefaultCharacter``` would need to be changed. That is done within the ```getCharInfo()``` function, shown here:

```
	#if __has_include ( <avr/pgmspace.h> )     
		void hyperdisplay::getCharInfo(uint8_t character, char_info_t * character_info) 
		{
			// This is the most basic font implementation, it only prints a monochrome character using the first color of the current window's current color sequence
			// If you want any more font capabilities then you should re-implement this function :D

			character_info->data = NULL;	// Use the window's current color

			// Link the default cordinate arrays
			character_info->xLoc = hyperdisplayDefaultXloc;
			character_info->yLoc = hyperdisplayDefaultYloc;

			character_info->xDim = 5;
			character_info->yDim = 8;

			// Figure out if the character should cause a newline
			if (character == '\r' || character == '\n')
			{
				character_info->causesNewline = true;
			}
			else
			{
				character_info->causesNewline = false;
			} 

			// Figure out if you need to actually show the chracter
			if((character >= ' ') && (character <= '~'))
			{
				character_info->show = true;
			}
			else
			{
				character_info->show = false;
				return;								// No point in continuing;
			}

			// Load up the character data and fill in coordinate data
			uint8_t values[5];							// Holds the 5 bytes for the character
			uint16_t offset = 6 + 5 * (character - 0);
			character_info->numPixels = 0;
			uint16_t n = 0;
			for(uint8_t indi = 0; indi < 5; indi++)
			{
				values[indi] = pgm_read_byte(font5x7 + offset + indi);
				for(uint8_t indj = 0; indj < 8; indj++)
				{
					if(values[indi] & (0x01 << indj))
					{
						character_info->numPixels++;
						*(character_info->xLoc + n) = (hd_font_extent_t)indi;
						*(character_info->yLoc + n) = (hd_font_extent_t)indj;
						n++;
					}
				}
			}
		}
	#else
		#warning "HyperDisplay Default Font Not Supported. Printing will not work without a custom implementation"
		void hyperdisplay::getCharInfo(uint8_t character, char_info_t * character_info) 
		{
			// A blank implementation of getCharInfo for when the target does not support <avr/pgmspace.h>
			character_info->show = false;
		}
	#endif /* __has_include( <avr/pgmspace.h> ) */
```
As you can see there are two versions of ```getCharInfo()``` -- one for when the platform supports 'avr/pgmspace.h' and one for when it does not. Generally I'm not a fan of conditional preprocessor options like this but it is important to have a default font so we've included it. This brings us to the second portion of the formula -- interpreting the MicroView font in 'font5x7.h'

### Interpreting MicroView 'font5x7.h'
Refer to the ```getCharInfo()``` function above. In order the steps it takes are:

1. Setting the ```data``` member to NULL - this signals to HD that we want to use the window's current color
1. Pointing at some memory locations that are provided to keep up to 5x7 pixels worth of x and y coordinates
1. Setting the dimensions of the character (y is set to 8 to leave a blank row of pixels between rows of text)
1. Setting the ```causesNewline``` member according to some basic rules
1. Setting the ```show``` member for visible characters
1. Setting the pixel coordinates and the number of pixels that need to be shown for this character from the information in 'font5x7.h'
    1. Make a 5-byte array to temporarily hold char data
    1. Set an offset into the 5x7 font array. 6 bytes are used for a header, and there are 5 bytes for every character after that. Since this map has values for all 255 standard ascii values we don't need to use an offset to ```character``` (hence ```(character - 0)```)
    1. Set ```numPixels``` to zero to start
    1. Loop through each byte in the font map
        1. Read the byte from the font map
        1. Loop through each bit in the byte
            1. If the bit is set to ```1``` then record that as a location in the ```character_info``` structure and increment the number of pixels that HD should render

Basically what those steps mean is that [each byte in the 5x7 font map represents 1/5 vertical columns of an 8-pixel tall character}(https://learn.sparkfun.com/tutorials/microview-hookup-guide/creating-fonts-for-microview). That mapping obviously can't apply directly to the 8x16 font - more likely that some pair of two bytes will represent 1/8 vertical column of the 16-pixel tall character. I can't say for sure which two bytes that would be... I vaguely remember that the top and bottom halves might be separated by a great distance within the font map. I don't know of a good place to find this information out other than just drawing it out on graph paper (that's what I had to do when I was looking at these fonts as well)





## Other Ways to Custom Font in HyperDisplay

HyperDisplay is meant to be very flexible, so there are several ways that you can implement a custom font. 

Before working on HyperDisplay I tried out some similar ideas in the RGB OLED Arduino Library. You can [check out the custom font example here](https://github.com/sparkfun/SparkFun_RGB_OLED_64x64_Arduino_Library/tree/master/examples/Example6_CustomFont). My favorite font was the 'QR Code Font' which took the 8-bit number for a given character and displayed it as a 2x4 area of blocks, and cycled through the rainbow as you typed. 

### Custom Font with Character Info

You can change ```getCharInfo()``` to change what pixels are shown where for a given character. One example I could imagine is printing a circle with radius equal to the character's 8-bit representation. Psuedocode might look like:
```
getCharInfo( character 'n' )
    set character dimensions to nxn
    use bresenham's circle algorithm to find coordinates of circle with radius=n
    set the locations of and number of pixels
```
And then the default HyperDisplay ```write()``` function could draw it for you. 

**Note to self:** Current default ```write()``` doesn't seem to handle color sequences quite like the rest of HyperDisplay. This could be a ToDo.

### Custom Font by Overwriting ```write()```

There is no reason that you need to use the default ```write()``` function at all! When you create a child class (that inherits from HyperDisplay) you could write a new ```write()``` function like so: (psuedocode)

```
write( character n )
    switch( n )
        case 'A' : 
            draw left line of 'A' /
            draw right line of 'A' \
            draw crossbar of 'A' -
            break;
        default: do nothing
```
or maybe
```
write( character n )
    fill some small area with HSV color where H = n/255
```

Of course these are oversimplified... ```write()``` handles positioning of the character within the screen as well. But there are so many possibilities!
