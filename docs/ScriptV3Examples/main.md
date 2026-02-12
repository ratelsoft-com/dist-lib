# main

Source: RatelWPF\Scripts\main.scr

```rscript
; main script sample (v3)
strict true
run_once "interlock_template.scr"

let axis = "Y"

servo_on(axis)
let start = get_pos(axis)
print("StartPos={}", start)

@interlock_run = 1
#let ih = spawn interlock_loop(axis, 500)
start_move_abs(axis, 1000)
sleep(100)
while(is_moving(axis) == 1)
{
    print("CurPos = {}", get_pos(axis))
    sleep(100)
}

let endpos = get_pos(axis)
print("stop pos={endpos}")
@interlock_run = 0
#await ih
assert(endpos >= start, "move failed")
end

```
