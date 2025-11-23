/* Alarm Project – (Unified)
   ATmega16 – AVR-GCC
   LCD + Keypad + Password Set/Check
*/

#define F_CPU 1000000UL
#include <avr/io.h>
#include <util/delay.h>

// LCD on PORTC (4-bit)
#define LCD_PORT PORTC
#define LCD_DDR  DDRC
#define RS 0
#define EN 1
#define D4 2
#define D5 3
#define D6 4
#define D7 5

// Keypad on PORTA
#define KEYPAD_PORT PORTA
#define KEYPAD_PIN  PINA
#define KEYPAD_DDR  DDRA

// --------------------------------------------------
// LCD FUNCTIONS
// --------------------------------------------------
void lcd_pulse()
{
    LCD_PORT |= (1<<EN);
    _delay_us(2);
    LCD_PORT &= ~(1<<EN);
    _delay_us(80);
}

void lcd_write4(uint8_t x)
{
    (x & 1) ? (LCD_PORT |= (1<<D4)) : (LCD_PORT &= ~(1<<D4));
    (x & 2) ? (LCD_PORT |= (1<<D5)) : (LCD_PORT &= ~(1<<D5));
    (x & 4) ? (LCD_PORT |= (1<<D6)) : (LCD_PORT &= ~(1<<D6));
    (x & 8) ? (LCD_PORT |= (1<<D7)) : (LCD_PORT &= ~(1<<D7));
    lcd_pulse();
}

void lcd_cmd(uint8_t c)
{
    LCD_PORT &= ~(1<<RS);
    lcd_write4(c>>4);
    lcd_write4(c & 0x0F);
    _delay_ms(2);
}

void lcd_putc(char c)
{
    LCD_PORT |= (1<<RS);
    lcd_write4(c>>4);
    lcd_write4(c & 0x0F);
    LCD_PORT &= ~(1<<RS);
    _delay_ms(1);
}

void lcd_puts(const char *s)
{
    while (*s) lcd_putc(*s++);
}

void lcd_init()
{
    LCD_DDR |= (1<<RS)|(1<<EN)|(1<<D4)|(1<<D5)|(1<<D6)|(1<<D7);
    _delay_ms(50);

    lcd_write4(0x03); _delay_ms(5);
    lcd_write4(0x03); _delay_ms(5);
    lcd_write4(0x03); _delay_ms(5);
    lcd_write4(0x02);

    lcd_cmd(0x28);
    lcd_cmd(0x0C);
    lcd_cmd(0x06);
    lcd_cmd(0x01);
}

// --------------------------------------------------
// KEYPAD FUNCTION
// --------------------------------------------------
char keypad_scan()
{
    const char keys[4][4] = {
        {'1','2','3','A'},
        {'4','5','6','B'},
        {'7','8','9','C'},
        {'*','0','#','D'}
    };

    for (uint8_t col=0; col<4; col++)
    {
        KEYPAD_DDR = 0xF0;
        KEYPAD_PORT = 0x0F;

        KEYPAD_DDR |= (1 << (col+4));
        KEYPAD_PORT &= ~(1 << (col+4));
        _delay_us(50);

        uint8_t row = (~KEYPAD_PIN) & 0x0F;

        if (row)
        {
            _delay_ms(20);
            row = (~KEYPAD_PIN) & 0x0F;

            for (uint8_t r=0; r<4; r++)
                if (row & (1<<r))
                    return keys[r][col];
        }
    }
    return 0;
}

int main(void)
{
    // Keypad init
    KEYPAD_DDR = 0xF0;
    KEYPAD_PORT = 0x0F;

    // LCD init
    lcd_init();

    // --------------------- (LCD + Keypad Test) ---------------------
    lcd_puts("Keypad + LCD OK");
    _delay_ms(1200);
    lcd_cmd(0x01);

    // ---------------------  (Password Logic) ---------------------
    char password[4];
    char input[4];

    // ---- Step 1: Set password ----
    lcd_cmd(0x01);
    lcd_puts("Set Pass:");

    for (uint8_t i=0; i<4; i++)
    {
        char k = 0;
        while (!k) k = keypad_scan();  // wait for key
        password[i] = k;
        lcd_putc('*');
        _delay_ms(300);
    }

    _delay_ms(800);
    lcd_cmd(0x01);

    // ---- Step 2: Ask for input ----
    lcd_puts("Enter Pass:");

    for (uint8_t i=0; i<4; i++)
    {
        char k = 0;
        while (!k) k = keypad_scan();
        input[i] = k;
        lcd_putc('*');
        _delay_ms(300);
    }

    lcd_cmd(0x01);

    // ---- Step 3: Compare ----
    uint8_t match = 1;
    for (uint8_t i=0; i<4; i++)
        if (input[i] != password[i])
            match = 0;

    if (match)
        lcd_puts("Correct!");
    else
        lcd_puts("Wrong!");

    while(1);
}
