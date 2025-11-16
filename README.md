# Alarm-project-

/*  alarm_project_basic_start.c
   Step 1: LCD + Keypad test
   ATmega16 – AVR-GCC
   This is just the first step of the alarm system project.
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

// --- LCD functions ---
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

// --- Keypad ---
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

// --- MAIN ---
int main(void)
{
    KEYPAD_DDR = 0xF0;
    KEYPAD_PORT = 0x0F;

    lcd_init();
    lcd_puts("Keypad + LCD");
    _delay_ms(1000);
    lcd_cmd(0x01);

    while (1)
    {
        char k = keypad_scan();
        if (k)
        {
            lcd_cmd(0x01);
            lcd_putc(k);
            _delay_ms(250);
        }
    }
}
