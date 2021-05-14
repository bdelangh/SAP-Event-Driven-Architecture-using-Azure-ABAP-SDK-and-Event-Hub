# SAP Event Driven Architecture using Azure ABAP SDK and Azure Event Hub

## Prerequisites
The Azure ABAP SDK is installed within an SAP system using [ABAPGit](https://docs.abapgit.org/). First you need to install ABAPGit. See [ABAPGit Installation](https://docs.abapgit.org/guide-install.html). It's recommended to install the developer edition.
Afterwards the ABAP SDK can be installed. See the installation section of [ABAP SDK for Azure](https://github.com/Microsoft/ABAP-SDK-for-Azure/blob/master/ABAP%20SDK%20for%20Azure%20-%20Github.pdf).

Once the installation is done, you can proceed with the creation of an Azure Eventhub and configure the ABAP SDK to connect to it. This is described at [ABAP SDK Implementation Guide for Azure Event hubs](https://github.com/microsoft/ABAP-SDK-for-Azure/blob/master/ABAP%20SDK%20Implementation%20Guide%20for%20Azure%20Event%20hubs%20.pdf). At the end use the test program `ZADF_DEMO_AZURE_EVENTHUB` to check if the communication with Azure EventHub is working successfully. Use transaction `ZREST_UTIL`to verify this.

<img src="images\zrestutiltest.jpg">

## Implementation

The text below focusses on a sample implementation for the SAP Bsuiness Partner Object (BUS1006).

### Event Type Linkage
With this setup done we can now plug the ABAP SDK into the `Workflow Event Framework`.
Got to transaction `SWETYPV - Type Linkages`. In this transaction you can create the link between workflow events and the receivers.

<img src="images\EvenTypeLinkages.jpg">

In the screen shot above you see the events for Business Object `BUS1006 - Business Partner`. The Receiver Type `BEH` is used for the SAP BPT Enterprise Messaging and the Receiver Type `AZEH` is newly created for events send to be send to Azure Event Hub using the Azure ABAP SDK.

Selecting the details reveals how the setup is done.

<img src="images\eventSetup.jpg">

The screens defines the event handler. In this case the handler is mapped to a static method of a custom class `ZBD_AZEVH_BUSPARTNER`. The prerequisite for such a custom is to implement the interface `BI_EVENT_HANDLER_STATIC`. The static `ON_EVENT` method of this interface will be called to handle the event. This method will contain the call to the ABAP SDK.

>Note : The `Linkage Activated` flag needs to be checked for the linkage to be activated.

### Event Handler Class
Sample code for the `ON_EVENT` method can be found beneath. The input parameters of the method contain the business object type, the business object id and event type (create, changed, deleted, ...).
I used the business object id to lookup some additional information on the business partner and put this into the message.
Error handling was done in a similar way to the SAP BPT Enterprise messaging via the SAP Application log. I even re-used the same method `cl_beh_application_log=>create` for this.
>Note : if you'd like to have the message body to adhere to the [Cloud Events](https://cloudevents.io/) standard as used by SAP BPT Enterprise messaging then you might also be able to re-use some code.
>Note : You'll probably need to rename the constants in the ABAP code below to the constants in your application.
    gc_interface_id = interface id of the ABAP SDK
    gc_message_class = message class for error message
    gc_appl_log_obj = SAP Application Log Object
    gc_appl_log_subobj = SAP Application Log Sub Object

```
  method BI_EVENT_HANDLER_STATIC~ON_EVENT.

    "InterfaceId to be used by ABAP SDK
    constants: gc_interface_id type zinterface_id value 'AZEHUBBDL',
               gc_message_class type symsgid value 'ZAZEH',
               gc_appl_log_obj type balobj_d value 'ZBD_AZEH',
               gc_appl_log_subobj type balsubobj value 'ZBUPA'.

    TYPES: BEGIN OF lty_message,
         busobj     TYPE sbo_bo_type,
         busobjname TYPE SBEH_BOTYP_TEXT,
         event      TYPE SIBFEVENT,
         date       type dats,
         time       type tims,
         objkey     TYPE SIBFBORIID,
         firstname  TYPE bu_namep_f,
         lastname   type bu_namep_l,
       END OF lty_message.

    data: lv_busobj_type        type sbo_bo_type,
          lv_busobj_type_name   type SBEH_BOTYP_TEXT,
          lv_buspartner_id      type BU_PARTNER,
          lv_centraldata_person type BAPIBUS1006_CENTRAL_PERSON,
          lv_firstname          type bu_namep_f,
          lv_lastname           type bu_namep_l.

    data: it_headers            TYPE tihttpnvp,
          wa_headers            TYPE LINE OF tihttpnvp,
          lv_error_string       TYPE string,
          lv_response           TYPE string,
          cx_interface          TYPE REF TO zcx_interace_config_missing,
          cx_http               TYPE REF TO zcx_http_client_failed,
          cx_adf_service        TYPE REF TO zcx_adf_service,
          oref_eventhub         TYPE REF TO zcl_adf_service_eventhub,
          oref                  TYPE REF TO zcl_adf_service,
          filter                type zbusinessid,
          lv_http_status        TYPE i, "HTTP Status code
          lo_json               TYPE REF TO cl_trex_json_serializer,
          lv_json_string        TYPE string,
          lv_json_xstring       TYPE xstring,
          lv_message            TYPE lty_message.

    data: lt_msg                TYPE bal_tt_msg,
          ls_msg                TYPE bal_s_msg.

    "select the business object type
    SELECT SINGLE bo_type INTO lv_busobj_type
      FROM sbo_i_bodef
      WHERE object_name = sender-typeid
      AND   object_type_category = sender-catid.
    IF sy-subrc = 0.
      SELECT SINGLE bo_text INTO lv_busobj_type_name
        FROM sbo_i_botypet
        WHERE bo_type  = lv_busobj_type
        AND   bo_textlan = sy-langu.
    ENDIF.

    lv_buspartner_id = sender-instid.

    "Get the BusinessPartner Details
    call function 'BAPI_BUPA_CENTRAL_GETDETAIL'
      exporting
         businesspartner = lv_buspartner_id
      importing
         centraldataperson = lv_centraldata_person.

    "Create the message
    lv_message-busobj     = sender-typeid.
    lv_message-busobjname = lv_busobj_type_name.
    lv_message-event      = event.
    lv_message-objkey     = lv_buspartner_id.
    lv_message-date       = sy-datlo.
    lv_message-time       = sy-timlo.
    lv_message-firstname  = lv_centraldata_person-firstname.
    lv_message-lastname   = lv_centraldata_person-lastname.

    TRY.
      "Calling Factory method to instantiate eventhub client

      oref = zcl_adf_service_factory=>create( iv_interface_id        = gc_interface_id
                                              iv_business_identifier = filter ).
      oref_eventhub ?= oref. "Type Cast to EventHub Object?

      "Setting Expiry time
      CALL METHOD oref_eventhub->add_expiry_time
        EXPORTING
          iv_expiry_hour = 0
          iv_expiry_min  = 15
          iv_expiry_sec  = 0.

      "Convert to JSON
      CREATE OBJECT lo_json
        EXPORTING
          data = lv_message.

      lo_json->serialize( ).
      lv_json_string  = lo_json->get_data( ).


      "Convert input string data to Xstring format
      CALL FUNCTION 'SCMS_STRING_TO_XSTRING'
        EXPORTING
          text   = lv_json_string
        IMPORTING
          buffer = lv_json_xstring
        EXCEPTIONS
          failed = 1
          OTHERS = 2.
      IF sy-subrc <> 0.
      ENDIF.

      " Sending Converted SAP data to Azure Eventhub
      CALL METHOD oref_eventhub->send
        EXPORTING
          request        = lv_json_xstring  "Input XSTRING of SAP Business Event data
          it_headers     = it_headers  "Header attributes
        IMPORTING
          response       = lv_response       "Response from EventHub
          ev_http_status = lv_http_status.   "Status

    CATCH zcx_interace_config_missing INTO cx_interface.
      lv_error_string = cx_interface->get_text( ).
      ls_msg-msgty  = 'E'.
      ls_msg-msgid  = gc_message_class.
      ls_msg-msgno  = '001'.
      ls_msg-msgv1  = sender-typeid.
      ls_msg-msgv2  = lv_buspartner_id.
      ls_msg-msgv3  = lv_error_string.
      APPEND ls_msg TO lt_msg.
      CALL METHOD cl_beh_application_log=>create
         EXPORTING
           i_msg       = lt_msg
           i_object    = gc_appl_log_obj
           i_subobject = gc_appl_log_subobj.

    CATCH zcx_http_client_failed INTO cx_http .
      lv_error_string = cx_http->get_text( ).
      ls_msg-msgty  = 'E'.
      ls_msg-msgid  = gc_message_class.
      ls_msg-msgno  = '001'.
      ls_msg-msgv1  = sender-typeid.
      ls_msg-msgv2  = lv_buspartner_id.
      ls_msg-msgv3  = lv_error_string.
      APPEND ls_msg TO lt_msg.
      CALL METHOD cl_beh_application_log=>create
         EXPORTING
           i_msg       = lt_msg
           i_object    = gc_appl_log_obj
           i_subobject = gc_appl_log_subobj.

    CATCH zcx_adf_service INTO cx_adf_service.
      lv_error_string = cx_adf_service->get_text( ).
      lv_error_string = cx_interface->get_text( ).
      ls_msg-msgty  = 'E'.
      ls_msg-msgid  = gc_message_class.
      ls_msg-msgno  = '001'.
      ls_msg-msgv1  = sender-typeid.
      ls_msg-msgv2  = lv_buspartner_id.
      ls_msg-msgv3  = lv_error_string.
      APPEND ls_msg TO lt_msg.
      CALL METHOD cl_beh_application_log=>create
         EXPORTING
           i_msg       = lt_msg
           i_object    = gc_appl_log_obj
           i_subobject = gc_appl_log_subobj.

    ENDTRY.

  endmethod.

```

### Surrounding Objects
The code above makes use of the following ABAP Objects :
* Message Clas : for error messages : use Transaction `SE91`

<img src="images\messages.jpg">

* Application Log Object and SubObject : use Transaction `SLG0`
<img src="images\slg0_1.jpg">

<img src="images\slg0_2.jpg">

### Testing
Testing can be done by either :
* Simulation of Workflow Event using transaction `SWUE`

<img src="images\simulateEvent.jpg">

* Or by changing a Business Partner using transaction `bp`

The resulting message can be seen in the ABAP SDK Monitoring transaction `ZREST_UTIL`

<img src="images\zrestutilt2.jpg">