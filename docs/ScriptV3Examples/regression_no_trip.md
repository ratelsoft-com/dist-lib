# regression_no_trip

Source: RatelWPF\Scripts\regression_no_trip.scr

```rscript
; regression: normal completion without interlock trip
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
let ih = spawn check_interlock(axis, 5000)
start_move_abs(axis, 500)

while(is_moving(axis) == 1 && is_running(ih) == 1)
{
    print("CurPos={}", get_pos(axis))
    sleep(50)
}

let endpos = get_pos(axis)
print("stop pos={endpos}")
stop(axis)
end

```
