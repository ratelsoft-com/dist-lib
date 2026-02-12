# interlock_template

Source: RatelWPF\Scripts\interlock_template.scr

```rscript
; interlock template (v3)
; include this file, then call check_interlock(axis, max_pos) in your main loop.
strict true
;let pos = get_pos("Y")
;print ("start pos={pos}")

while(1)
{
    let cur_pos = get_pos("Y")

    if (cur_pos > 500)
    {
        all_stop()
        trip("Interlock trip: axis=Y, pos={cur_pos}, limit=500")
    }
    sleep(100)
    
}
end

```
