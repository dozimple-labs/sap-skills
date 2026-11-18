# Replacements for NetWeaver 7.50

Examples of code written for later releases and its 7.50 equivalent. Names are generic.

## ENUM → constants

```abap
" 7.51+
TYPES: BEGIN OF ENUM ty_status, open, released, closed, END OF ENUM ty_status.

" 7.50
INTERFACE lif_status.
  CONSTANTS: open     TYPE c LENGTH 1 VALUE 'O',
             released TYPE c LENGTH 1 VALUE 'R',
             closed   TYPE c LENGTH 1 VALUE 'C'.
ENDINTERFACE.
```

If the values are stored in a table, prefer a domain with fixed values: the F4 help and the
documentation come with it.

## SELECT FROM @itab → FOR ALL ENTRIES

```abap
" 7.52+
SELECT a~matnr, a~maktx
  FROM @lt_keys AS k
  INNER JOIN makt AS a ON a~matnr = k~matnr
  WHERE a~spras = @sy-langu
  INTO TABLE @DATA(lt_texts).

" 7.50
IF lt_keys IS NOT INITIAL.          " an empty driver table selects EVERYTHING
  SELECT matnr, maktx
    FROM makt
    FOR ALL ENTRIES IN @lt_keys
    WHERE matnr = @lt_keys-matnr
      AND spras = @sy-langu
    INTO TABLE @DATA(lt_texts).
ENDIF.
```

`FOR ALL ENTRIES` removes duplicate rows from the result. Select every key field you need to keep rows
apart, and do not combine it with `GROUP BY`.

## Common table expression → two steps or a CDS view

```abap
" 7.51+
WITH +orders AS ( SELECT vbeln FROM vbak WHERE erdat >= @lv_from )
SELECT p~vbeln, p~posnr, p~matnr
  FROM vbap AS p INNER JOIN +orders AS o ON o~vbeln = p~vbeln
  INTO TABLE @DATA(lt_items).

" 7.50: the same as a join, or a subquery
SELECT p~vbeln, p~posnr, p~matnr
  FROM vbap AS p
  WHERE p~vbeln IN ( SELECT vbeln FROM vbak WHERE erdat >= @lv_from )
  INTO TABLE @DATA(lt_items).
```

When the same intermediate result is reused in several programs, a CDS view is the cleaner 7.50 answer.

## SQL string and date functions → ABAP after the SELECT

```abap
" 7.51+
SELECT kunnr, upper( name1 ) AS name FROM kna1 INTO TABLE @DATA(lt_c).

" 7.50
SELECT kunnr, name1 FROM kna1 INTO TABLE @DATA(lt_c).
LOOP AT lt_c ASSIGNING FIELD-SYMBOL(<c>).
  <c>-name1 = to_upper( <c>-name1 ).
ENDLOOP.
```

Dates: `lv_due = lv_date + 30.` and `lv_days = lv_to - lv_from.` work on type `d` fields directly.

## AVG with type → SUM and COUNT

```abap
" does not compile on 7.50
SELECT matnr, AVG( menge AS DEC( 13, 3 ) ) AS avg_qty FROM ekpo GROUP BY matnr INTO TABLE @DATA(lt_a).

" 7.50
SELECT matnr, SUM( menge ) AS total, COUNT(*) AS cnt FROM ekpo GROUP BY matnr INTO TABLE @DATA(lt_s).
LOOP AT lt_s ASSIGNING FIELD-SYMBOL(<s>).
  DATA(lv_avg) = COND decfloat34( WHEN <s>-cnt > 0 THEN <s>-total / <s>-cnt ).
ENDLOOP.
```

## LOOP STEP → index arithmetic

```abap
" 7.57+
LOOP AT lt_rows INTO DATA(ls_row) STEP 2.
ENDLOOP.

" 7.50
DATA(lv_idx) = 1.
WHILE lv_idx <= lines( lt_rows ).
  DATA(ls_row) = lt_rows[ lv_idx ].
  lv_idx = lv_idx + 2.
ENDWHILE.
```

## utclong → timestampl

```abap
" 7.54+
DATA(lv_now) = utclong_current( ).

" 7.50
DATA lv_now TYPE timestampl.
GET TIME STAMP FIELD lv_now.
```

## boolc → xsdbool

```abap
DATA lv_flag TYPE abap_bool.
lv_flag = boolc( lv_qty > 0 ).    " returns a string: compare and pass it with care
lv_flag = xsdbool( lv_qty > 0 ).  " returns abap_bool
```
