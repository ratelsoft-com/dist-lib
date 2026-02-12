# mainscr

Source: RatelWPF\Scripts\mainscr.scr

```rscript
; Script v3 sample (include + subroutine + async)
;include "interlock_y_limit.scr"
let axis = "Y"
run_interlock=1
func check_interlock(axis, max_pos)
{
while(run_interlock)
{
  let curpos = get_pos(axis)
  if (curpos > max_pos)
  {
    trip("Interlock trip")
  }
  sleep(50)
}
}


let ih=spawn check_interlock(axis,400)

servo_on(axis)
start_move_abs(axis, 1000)
sleep(100)
while(is_moving(axis) == 1) {
;    call interlock_y_limit()
  print ("CurPos = {}", get_pos(axis))
  sleep(100)
}
print ("stop pos={}", get_pos(axis))

move_abs(axis, 1200)
print ("last pos={}", get_pos(axis)) 
run_interlock=0
await ih
end
```
