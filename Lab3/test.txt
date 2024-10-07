#include <avr/io.h>
#include <util/delay.h>

#define SCK PB5
#define SDI PB3
#define LATCH PB2
#define JUMP_BUTTON PB1  // Button pin

uint8_t skey;
uint8_t vbuf[8];  // LED matrix buffer

void spi_transfer(uint8_t data);
void max7219_sendData(uint8_t addr, uint8_t data);
void SPI_MasterInit(void);
void max7219_Init(void);
void max7219_Update(void);
void max7219_Pixel(uint8_t x, uint8_t y, uint8_t p);
void DinoJump(void);
uint8_t DetectCollision(uint8_t dino_x, uint8_t dino_y, uint8_t obstacle_x, uint8_t obstacle_y);
uint8_t ButtonPressed(void);

void spi_transfer(uint8_t data) 
{
    uint8_t i;
    for(i = 0; i < 8; i++) {
        if(data & 0x80) {
            PORTB |= 1 << SDI;
        } else {
            PORTB &= ~(1 << SDI);
        }
        data <<= 1;
        PORTB |= 1 << SCK;
        _delay_us(1);
        PORTB &= ~(1 << SCK);
        _delay_us(1);
    }
}

// Send data to MAX7219
void max7219_sendData(uint8_t addr, uint8_t data)
{
    PORTB &= ~(1 << PB2);
    // SPDR = addr;
    // while (!(SPSR & (1<<SPIF)));
    // SPDR = data;
    // while (!(SPSR & (1<<SPIF)));
    spi_transfer(addr);
    spi_transfer(data);
    PORTB |= (1 << PB2);
}

void SPI_MasterInit(void)
{
    DDRB |= (1<<PB3)|(1<<PB5);
    // SPCR = (1<<SPE)|(1<<MSTR)|(0<<SPR0);
}


void max7219_Init(void) 
{
    DDRB |= 1 << PB2; // CS as output
    max7219_sendData(0x0F, 0); // Normal mode
    max7219_sendData(0x09, 0); // No decode
    max7219_sendData(0x0B, 7); // 8 digits
    max7219_sendData(0x0A, 10); // Intensity
    max7219_sendData(0x0C, 1); // Power on
    // Clear display
    for (uint8_t i = 0; i < 8; i++) {
        max7219_sendData(i+1, 0);
    }
}

void max7219_Update(void) 
{
    for (uint8_t i = 0; i < 8; i++) {
        max7219_sendData(i+1, vbuf[i]);
    }
}

void Max7219_SetPixel(uint8_t x, uint8_t y, uint8_t p) 
{
    if (p == 0) {
        vbuf[y] &= ~(1 << x);
    } else {
        vbuf[y] |= (1 << x);
    }
    max7219_Update();
}

uint8_t ButtonPressed(void) 
{
    static uint8_t shreg;
    shreg <<= 1;
    if((PINB & (1 << PB1)) != 0) {
    shreg |= 1;
    }
    if((shreg & 0x07) == 0x04) {
        skey= 1;
    }
}   

void gameInit(void)
{
    for (uint8_t i = 0; i < 8; ++i)
    {
        Max7219_SetPixel(i, 7, 1);
    }
    Max7219_SetPixel(3, 6, 1);
}
   

void gameEngine(void)  //--- 25 ms
{
 static unsigned char sg = 0, gg = 0, isrun = 1;
 static unsigned char dl1 = 8, h = 7, dl2 = 3, v=25;
 
 if(isrun)
 {
  // move kaktus -----------
  if(dl1 > 0)
  {
    --dl1;
  }
  else
  {
   dl1 = v;

    Max7219_SetPixel(h, 6, 0);
    if(h > 0) --h;
     else
     {
       h= 7;
    //    PrintResult(++gg);
       if((v > 5) && (gg > 5)) --v;
    //    Beep(5);
     }
   
    // collision detect ----------
    if(h == 3)
    {
      if(sg == 0 || sg == 9)
      {
        isrun = 0;
        // Beep(250);
      }
    }
    Max7219_SetPixel(h, 6, 1);
  }
 
  //--------------------------------
  if(skey == 1)
  {
    if(dl2 > 0)
    {
     --dl2;
    }
    else
    {
     dl2= 8;
     switch(sg)
     {
       case 0:
          Max7219_SetPixel(3, 6, 0);
          Max7219_SetPixel(3, 5, 1);
          sg = 1;
        break;
       case 1:
          Max7219_SetPixel(3, 5, 0);
          Max7219_SetPixel(3, 4, 1);
          sg =2;
        break;
      case 2:
          Max7219_SetPixel(3, 4, 0);
          Max7219_SetPixel(3, 3, 1);
          sg =3;
        break;
      case 3:
          Max7219_SetPixel(3, 3, 0);
          Max7219_SetPixel(3, 2, 1);
          sg =4;
        break;
      case 4:
          Max7219_SetPixel(3, 2, 0);
          Max7219_SetPixel(3, 1, 1);
          sg =5;
        break;
      case 5:
          Max7219_SetPixel(3, 1, 0);
          Max7219_SetPixel(3, 2, 1);
          sg =6;
        break;
      case 6:
          Max7219_SetPixel(3, 2, 0);
          Max7219_SetPixel(3, 3, 1);
          sg =7;
        break;
      case 7:
          Max7219_SetPixel(3, 3, 0);
          Max7219_SetPixel(3, 4, 1);
          sg =8;
        break;
      case 8:
          Max7219_SetPixel(3, 4, 0);
          Max7219_SetPixel(3, 5, 1);
          sg =9;
        break;
      case 9:
          Max7219_SetPixel(3, 5, 0);
          Max7219_SetPixel(3, 6, 1);
          sg =0;
          skey =0;
        break;
     }
    }
  }
}
else
{
  if (!isrun && skey)
  {
    skey = 0;
    gameInit();
   
    sg = 0;
    isrun = 1;
    dl1 = 8;
    v = 25;
    h = 7;
    dl2 = 2;
    gg = 0;
    // PrintResult(0);
  }
 }
}
int main(void) 
{
    SPI_MasterInit();
    max7219_Init();

    gameInit();

    for(;;)
    {
        ButtonPressed();
       gameEngine();
       _delay_ms(10);
    }
}
