from gpiozero import LED, Button
from time import sleep, time
import sys
import select
import signal
from RPi import GPIO

# === GPIO CONFIG ===
output_pins = {
    10: 17,  # Assign button GPIO10 → Output GPIO17
    9: 18,   # Assign button GPIO9  → Output GPIO18
    11: 19,  # Assign button GPIO11 → Output GPIO19
    8: 20,   # Assign button GPIO8  → Output GPIO20
}

reset_button_pin = 7
led_pin = 22  # LED for feedback (GPIO22)
stop_input_pin = 12  # GPIO12 used to stop outputs

# === INTERNAL STATE ===
barcode_pin_map = {}
gpio_objects = {}
assignment_inputs = {pin: Button(pin, pull_up=True, bounce_time=0.3) for pin in output_pins}
reset_button = Button(reset_button_pin, pull_up=True, bounce_time=0.3)
stop_button = Button(stop_input_pin, pull_up=True, bounce_time=0.3)
feedback_led = LED(led_pin)
assign_mode = None
active_output = None
output_start_time = None

# === FUNCTIONS ===
def blink_feedback():
    """Blink feedback LED for 0.5 sec"""
    feedback_led.on()
    sleep(0.5)
    feedback_led.off()

def switch_output(barcode):
    """Turn ON output and keep ON until stop condition"""
    global active_output, output_start_time
    short_code = barcode[:15]
    if short_code in barcode_pin_map:
        pin = barcode_pin_map[short_code]
        if pin not in gpio_objects:
            gpio_objects[pin] = LED(pin)
        print(f"[ACTION] Barcode {short_code} → GPIO{pin} ON")
        gpio_objects[pin].on()
        active_output = gpio_objects[pin]
        output_start_time = time()
    else:
        print(f"[WARN] Unknown barcode {short_code}")

def assign_barcode(short_code, pin):
    """Assign barcode to pin and blink feedback"""
    barcode_pin_map[short_code] = pin
    print(f"[ASSIGN] {short_code} → GPIO{pin}")
    blink_feedback()

def reset_assignments():
    """Clear all assignments and blink feedback"""
    barcode_pin_map.clear()
    print("[RESET] All assignments cleared")
    blink_feedback()

def read_barcode_nonblocking():
    """Check if barcode scanner sent data"""
    if select.select([sys.stdin], [], [], 0.0)[0]:
        return sys.stdin.readline().strip()
    return None

def cleanup_and_exit(*args):
    """Cleanup GPIO and exit safely"""
    print("\n[EXIT] Cleaning up GPIO...")
    for led in gpio_objects.values():
        led.off()
    feedback_led.off()
    GPIO.cleanup()
    sys.exit(0)

# === MAIN LOOP ===
def main():
    global assign_mode, active_output, output_start_time
    print("[READY] Press buttons or scan barcode. Ctrl+C to stop.")

    # Handle Ctrl+C or kill
    signal.signal(signal.SIGINT, cleanup_and_exit)
    signal.signal(signal.SIGTERM, cleanup_and_exit)

    while True:
        # Check assign buttons
        for btn_pin, btn in assignment_inputs.items():
            if btn.is_pressed:
                print(f"[BUTTON] Assign button GPIO{btn_pin} pressed")
                assign_mode = output_pins[btn_pin]
                sleep(0.3)  # debounce

        # Check reset button
        if reset_button.is_pressed:
            print(f"[BUTTON] Reset button GPIO{reset_button_pin} pressed")
            reset_assignments()
            assign_mode = None
            sleep(0.3)

        # Check barcode
        barcode = read_barcode_nonblocking()
        if barcode:
            short_code = barcode[:15]
            if assign_mode:
                assign_barcode(short_code, assign_mode)
                assign_mode = None
            else:
                switch_output(barcode)

        # Check stop input (GPIO12)
        if active_output and stop_button.is_pressed:
            if time() - output_start_time >= 20:  # 20 second timer
                print("[STOP] GPIO12 held → Turning OFF output")
                active_output.off()
                active_output = None
                output_start_time = None

        sleep(0.05)  # CPU friendly loop

if __name__ == "__main__":
    main()
