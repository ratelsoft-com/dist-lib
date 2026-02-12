# interlock_y_limit

Source: RatelWPF\Scripts\interlock_y_limit.scr

```rscript
; interlock library script (v3)
; standard template: check_interlock(axis, max_pos)
func check_interlock(axis, max_pos)
{
    let cur_pos = get_pos(axis)
    if (cur_pos > max_pos)
    {
        trip("Interlock trip: axis={axis}, pos={cur_pos}, limit={max_pos}")
    }

    return cur_pos
}

; backward-compatible helper
func interlock_y_limit(axis)
{
    return check_interlock(axis, 500)
}

```
