# ABAP_01
REPORT zabap_p16_01.

* Definição da Internal Table, baseado na definição do ZTY_FOCC, clique sobre esta definição
DATA: it_focc TYPE zty_voo.

* Definião da Work Área, para poder manipular os registros de uma tabela interna com mesmo layout
DATA: wa_focc  TYPE demo_focc,
      mark     TYPE c VALUE 'X',
      wa_sbook TYPE sbook.

* Tela de Filtro onde o usúuário poderá selecionar o que deseja processar
SELECTION-SCREEN BEGIN OF BLOCK abcd WITH FRAME TITLE text-001.
SELECT-OPTIONS: so_car FOR wa_focc-carrid OBLIGATORY,
                so_con FOR wa_focc-connid OBLIGATORY.
SELECTION-SCREEN END OF BLOCK abcd.

*O usuario poderá escolher entre 3 formas de visualizaçao 
SELECTION-SCREEN BEGIN OF BLOCK xyz WITH FRAME TITLE text-abc.
PARAMETERS: pa_all   RADIOBUTTON GROUP aaa,
            pa_menor RADIOBUTTON GROUP aaa,
            pa_maior RADIOBUTTON GROUP aaa.
SELECTION-SCREEN END OF BLOCK xyz.


SELECTION-SCREEN BEGIN OF SCREEN 1001.
SELECT-OPTIONS: so_pas FOR wa_sbook-bookid.
SELECTION-SCREEN END OF SCREEN 1001.

* Evento para consistir dados da tela de filtro
AT SELECTION-SCREEN on BLOCK abcd.
  AUTHORITY-CHECK OBJECT 'S_CARRID'
           ID 'CARRID' FIELD so_car-low
           ID 'ACTVT' FIELD '03'.
  IF sy-subrc <> 0.
* Implement a suitable exception handling here
    MESSAGE e010(zabap)
    WITH: sy-uname,
          so_car.
  ENDIF.

START-OF-SELECTION.
* Leitura da Tabela do Bco de Dados e carrega em memória na tabela interna IT_SFLIGHT
  SELECT carrid connid fldate seatsmax seatsocc  FROM sflight INTO TABLE it_focc
     WHERE carrid IN so_car
     AND   connid IN so_con.

  IF sy-subrc NE 0.
    MESSAGE i009(zabap) WITH so_car so_con. 
    EXIT.
  ENDIF.
  
* Processar todos os registros da Tabela Interna para efetuar o calculo de percentage
  LOOP AT it_focc INTO wa_focc.
    wa_focc-percentage = wa_focc-seatsocc * 100 /  wa_focc-seatsmax.
* Estou modificando o registro da tabela mas apenas regravando o campo Percentage
    MODIFY it_focc FROM wa_focc TRANSPORTING percentage.
  ENDLOOP.
* Classificação da Tabela Interna
  SORT it_focc BY percentage DESCENDING.
* Relatório resultante, após cálculo, regravação e classificação
  LOOP AT it_focc INTO wa_focc.
    CASE mark.
      WHEN pa_all.
        PERFORM listar.

      WHEN pa_menor.
        IF wa_focc-percentage < 50.
          PERFORM listar.
        ENDIF.
      WHEN OTHERS.
        IF wa_focc-percentage => 50.
          PERFORM listar.
        ENDIF.
    ENDCASE.
  ENDLOOP.
  CLEAR wa_focc-carrid.

* segunda tela  depois dos dois clicks
AT LINE-SELECTION.
  CASE sy-lsind.
    WHEN 1.
      CHECK wa_focc-carrid IS NOT INITIAL.
      CALL SELECTION-SCREEN 1001.
      CHECK sy-subrc = 0.
      SELECT * FROM sbook INTO wa_sbook WHERE carrid = wa_focc-carrid
                                        AND   connid = wa_focc-connid
                                        AND   fldate =  wa_focc-fldate
                                        AND   bookid IN so_pas.
        WRITE:/ wa_sbook-carrid, wa_sbook-connid, wa_sbook-fldate, wa_sbook-bookid, wa_sbook-CUSTOMID, wa_sbook-passname.
        hide:  wa_sbook-CUSTOMID.
      ENDSELECT.
      CLEAR wa_focc-carrid.
      CLEAR  wa_sbook-CUSTOMID.
      WHEN 2.
        CHECK  wa_sbook-CUSTOMID is not INITIAL.
        SUBMIT ZABAP_LER_SCUSTOM
                WITH PA_PASS =  wa_sbook-CUSTOMID via SELECTION-SCREEN AND RETURN.


    WHEN OTHERS.
      MESSAGE i011(zabap).

  ENDCASE.

*&---------------------------------------------------------------------*
*&      Form  LISTAR
*&---------------------------------------------------------------------*
*       text
*----------------------------------------------------------------------*
*  -->  p1        text
*  <--  p2        text
*----------------------------------------------------------------------*
FORM listar .
  WRITE:/ wa_focc-carrid, wa_focc-connid, wa_focc-fldate, wa_focc-seatsmax,
                  wa_focc-seatsocc, wa_focc-percentage COLOR = 6.
  HIDE:  wa_focc-carrid, wa_focc-connid, wa_focc-fldate.
ENDFORM.
