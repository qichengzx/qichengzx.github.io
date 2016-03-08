#SELECT * FROM `toupiao`.`tp_reply` WHERE 1=1 and `key` REGEXP '[[:<:]]是吧[[:>:]]' ORDER BY id desc LIMIT 0,5
#SELECT * FROM `toupiao`.`tp_reply` WHERE 1=1 and `key` REGEXP '[[:<:]]345[[:>:]]' ORDER BY id desc LIMIT 0,5
SELECT * FROM `toupiao`.`tp_reply` WHERE 1=1 and FIND_IN_SET('345',`key`)  ORDER BY id desc LIMIT 0,5
