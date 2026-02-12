# regression_trip_stop

Source: RatelWPF\Scripts\regression_trip_stop.scr

```rscript
; regression: interlock trip must stop script before next statements
strict true
let axis = "Y"

func check_interlock(axis, max_pos)
{
    while(1)
    {
        let cur_pos = get_pos(axis)
        if (cur_pos > max_pos)
        {
            trip("Regression trip: axis={axis}, pos={cur_pos}, limit={max_pos}")
        }
        sleep(20)
    }
}

servo_on(axis)
let ih = spawn check_interlock(axis, 400)
start_move_abs(axis, 1000)

while(is_moving(axis) == 1 && is_running(ih) == 1)
{
    print("CurPos={}", get_pos(axis))
    sleep(50)
}

; If trip works correctly, this line should NOT be printed.
print("UNEXPECTED: stop pos={}", get_pos(axis))
await ih
end

```
