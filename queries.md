query-moodle-2.x
================

Notas:
=======
Para sumar campos de tipo TIME.

SELECT SEC_TO_TIME(SUM(TIME_TO_SEC(`login`))) FROM Table1; 

referencia: http://stackoverflow.com/questions/2217139/mysql-average-on-time-column

Para restar campos tipo fecha: 
- http://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_timediff
- https://gonzalojpv.wordpress.com/2013/04/14/fechas-con-mysql/

Procesando el log de moodle
==============================

- Crear una vista para apoyarse en los reportes (view name: course_log_activities)
<blockquote>
SELECT
  log.id,
  log.time, 
  log.userid, 
  log.course, 
  c.fullname , 
  FROM_UNIXTIME(time)  as fecha ,
  DATE_FORMAT(FROM_UNIXTIME(time) , '%Y' ) as year,
  DATE_FORMAT(FROM_UNIXTIME(time) , '%m' ) as month,
  DATE_FORMAT(FROM_UNIXTIME(time) , '%d' ) as day,
  DATE_FORMAT(FROM_UNIXTIME(time) , '%H:%i:%S' ) as hour
FROM mdl_log log
INNER JOIN mdl_course c ON c.id = log.course
WHERE module = 'course'
GROUP BY course, userid, year, month, day,  hour
ORDER BY  course, userid, time, id  ASC
</blockquote>

- Para calcular la dedicación a partir de lecturas del log

SELECT 
  u.firstname as Nombre, 
  u.lastname as Apellido, 
  u.email as Email,  
  co.fullname as Curso,  
  tiempo.time as 'Dedicación (s)' 
FROM mdl_user u
INNER JOIN (
SELECT c.userid, c.course, SUM(TIME_TO_SEC(c.time)) as time FROM 
(SELECT b.userid, b.course, b.`year`,  SEC_TO_TIME(SUM(TIME_TO_SEC(b.time))) as time FROM
(SELECT id, userid, course, year, `month`,  SEC_TO_TIME(SUM(TIME_TO_SEC(time))) as time FROM
(SELECT c.id, c.userid, c.course, c.`year`, c.`month`, c.`day`, TIMEDIFF( MAX(c.`hour`) , MIN(c.`hour`) )as time 
FROM course_log_activities c
GROUP BY course, userid, `year`, `month`, `day`) dias
GROUP BY dias.course,  dias.userid, dias.`year`, dias.`month`) b
GROUP BY b.course, b.userid, b.`year`) c
WHERE c.time > 0
GROUP BY c.course, c.userid
) tiempo ON tiempo.userid = u.id
INNER JOIN mdl_course co ON co.id = tiempo.course
WHERE u.id<>'1'
ORDER BY co.fullname, u.id

- Relación de usuarios con su información, y los cursos matriculados.

SELECT  
  mue.enrolid,
  erol.enrol,
  erol.courseid,
  c.fullname,
  c.shortname,
  u.id as user_id, 
  u.username as login_name, 
  u.firstname as Nombre, 
  u.lastname as Apellido, 
  u.email as Email,  
  u.city as Ciudad,  
FROM_UNIXTIME(u.firstaccess) as "Primer Acceso", 
FROM_UNIXTIME(u.lastaccess) as "Ultimo Acceso", 
FROM_UNIXTIME(u.lastlogin) as "Ultimo login"
FROM  mdl_user_enrolments mue
INNER JOIN mdl_user u ON u.id = mue.userid
INNER JOIN mdl_enrol erol ON erol.id = mue.enrolid
INNER JOIN mdl_course c ON c.id = erol.courseid
ORDER BY erol.courseid

