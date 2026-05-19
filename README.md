- ### **Autor: Daniel Jimenez Ramirez**

- **Profesor: Willman Acosta Lugo**  
- **Actividad: AEE: Explotación Tecnológica en ERP/CRM**

---

### Entregable 1: Generación del Informe Dinámico (QWeb XML)

1. **Sintaxis estructurada:** El documento debe estar bien formado e incluir espacios de nombres correctos si fuera necesario.

El XML parece estar bien formado, aunque le añadí la cabecera para la declaración de la versión y el encoding usado.
```
<?xml version="1.0" encoding="UTF-8"?>
```
2. **Lógica embebida QWeb:** Debe hacer uso de directivas de iteración (t-foreach) para desglosar dinámicamente las líneas de la factura (invoice\_line\_ids).

Este XML ya hace uso de estas directivas. He añadido unos comentarios sabiendo que hace cada cosa.
```
<tbody class="invoice_tbody">
        <t t-set="lines" t-value="o.invoice_line_ids.sorted(key=lambda l: (-l.sequence, l.date, l.move_name, -l.id), reverse=True)"/>
        <!-- Se invoca un método del modelo (_get_move_lines_to_report) que filtra y
            ordena las líneas de la factura (invoice_line_ids) según reglas de negocio. -->
        <t t-set="lines_to_report" t-value="o._get_move_lines_to_report()"/>
        <t t-set="current_section" t-value="None"/>
        <t t-set="current_subsection" t-value="None"/>
        <!-- Itera sobre cada línea devuelta por el método anterior.
        Por cada iteración se determina el tipo de línea (display_type): line_section, line_subsection, line_note, product. -->
        <t t-foreach="lines_to_report" t-as="line">
            <t t-set="is_note" t-value="line.display_type == 'line_note'"/>
            <t t-set="is_section" t-value="line.display_type == 'line_section'"/>
            <t t-set="is_subsection" t-value="line.display_type == 'line_subsection'"/>
            <t t-set="is_product" t-value="line.display_type == 'product'"/>
            
            <t t-if="is_section">
                <t t-set="current_section" t-value="line"/>
                <t t-set="current_subsection" t-value="None"/>
                <t t-set="line_padding" t-value="0"/>
            </t>
            <t t-elif="is_subsection">
                <t t-set="current_subsection" t-value="line"/>
                <t t-set="line_padding" t-value="3"/>
            </t>
            <t t-else="">
                <t t-set="line_padding" t-value="4 if current_subsection else (3 if current_section else 0)"/>
            </t>
            <t t-set="padding_class" t-value="line_padding and ('px-' + str(line_padding)) or ''"/>
            <t t-set="hide_details" t-value="line.collapse_composition"/>
            <t t-set="hide_prices" t-value="line.collapse_prices"/>
            
            <t t-if="not hide_details and not hide_prices">
                <tr t-att-class="                                             'fw-bolder o_line_section' if is_section else                                             'fw-bold o_line_subsection' if is_subsection else                                             'fst-italic o_line_note' if is_note                                             else ''">
                    <t t-set="line_colspan" t-value="3 + (1 if display_discount else 0) + (1 if display_taxes and not line.tax_ids else 0)"/>
                    <t t-if="is_product" name="account_invoice_line_accountable">
                        <td name="account_invoice_line_name" t-att-class="padding_class">
                            <span t-if="line.name" t-field="line.name" t-options="{'widget': 'text'}" dir="auto">Bacon Burger</span>
                        </td>
                        <td name="td_quantity" class="o_td_quantity text-end">
                            <span t-field="line.quantity" class="text-nowrap">3.00</span>
                            <span t-field="line.product_uom_id" groups="uom.group_uom">units</span>
                            <span t-if="line.product_uom_id != line.product_id.uom_id" groups="uom.group_uom" class="text-muted small">
                                <br/>
                                <t t-set="product_uom_qty" t-value="line.product_uom_id._compute_quantity(line.quantity, line.product_id.uom_id)"/>
                                <span t-out="product_uom_qty" t-options="{'widget': 'float', 'decimal_precision': 'Product Unit'}" data-oe-demo="3.00"/> <span t-field="line.product_id.uom_id" data-oe-demo="units"/>
                            </span>
                        </td>
                        <td name="td_price_unit" t-attf-class="text-end {{ 'd-none d-md-table-cell' if report_type == 'html' else '' }}">
                            <span t-if="not hide_prices" class="text-nowrap" t-field="line.price_unit">9.00</span>
                        </td>
                        <td name="td_discount" t-if="display_discount" t-attf-class="text-end {{ 'd-none d-md-table-cell' if report_type == 'html' else '' }}">
                            <span class="text-nowrap" t-field="line.discount">0</span>
                        </td>
                        <t t-set="taxes" t-value="', '.join(tax.tax_label for tax in line.tax_ids if tax.tax_label)"/>
                        <td name="td_taxes" t-if="display_taxes" t-attf-class="text-end {{ 'd-none d-md-table-cell' if report_type == 'html' else '' }} {{ 'text-nowrap' if len(taxes) &lt; 10 else '' }}">
                            <span t-if="not hide_prices" t-out="taxes" id="line_tax_ids">Tax 15%</span>
                        </td>
                        <td name="td_subtotal" class="text-end o_price_total">
                            <span t-if="not hide_prices and o.company_price_include == 'tax_excluded'" class="text-nowrap" t-field="line.price_subtotal">27.00</span>
                            <span t-if="not hide_prices and o.company_price_include == 'tax_included'" class="text-nowrap" t-field="line.price_total">31.05</span>
                        </td>
                    </t>
                    <t t-elif="line.display_type in ('line_section', 'line_subsection')">
                        <t t-set="section_subtotal" t-value="line.get_section_subtotal()"/>
                        <!-- Para ordenadores -->
                        <td name="line_name_td" t-att-colspan="(1 if is_product else line_colspan)" t-attf-class="{{'d-none d-md-table-cell' if report_type == 'html' else ''}} {{padding_class}}">
                            <span t-if="line.name" t-field="line.name" t-options="{'widget': 'text'}" dir="auto">A section title</span>
                        </td>
                        <!-- Para moviles -->
                        <td colspan="2" t-attf-class="{{'d-md-none d-table-cell' if report_type == 'html' else 'd-none'}} {{padding_class}}">
                            <span t-if="line.name" t-field="line.name" t-options="{'widget': 'text'}" dir="auto">A section title</span>
                        </td>
                        <td colspan="1" class="text-end o_price_total">
                            <span t-out="section_subtotal" class="text-nowrap" t-options="{&quot;widget&quot;: &quot;monetary&quot;, &quot;display_currency&quot;: o.currency_id}"/>
                        </td>
                    </t>
                    <t t-elif="is_note">
                        <td colspan="99">
                            <span t-if="line.name" t-field="line.name" t-options="{'widget': 'text'}" dir="auto">A note, whose content usually applies to the section or product above.</span>
                        </td>
                    </t>
                </tr>
            </t>
            <t t-else="">
                <t t-set="grouped_lines" t-value="line._get_child_lines()"/>
                <!-- Este bucle despliega las líneas agrupadas de una línea que tiene
                    colapso activo, manteniendo la estructura pero omitiendo detalles de precios o cantidades según corresponda. -->
                <t t-foreach="grouped_lines" t-as="grouped_line">
                    <t t-if="grouped_line['display_type'] == 'product' or hide_details">
                        <t t-set="line_colspan" t-value="1"/>
                    </t>
                    <t t-else="">
                        <t t-set="line_colspan" t-value="3 + (1 if display_discount else 0) + (1 if display_taxes and not grouped_line.get('taxes') else 0)"/>
                    </t>
                    <t t-if="hide_prices">
                        <t t-if="grouped_line['display_type'] == 'line_section'">
                            <t t-set="current_section" t-value="grouped_line"/>
                            <t t-set="current_subsection" t-value="None"/>
                        </t>
                        <t t-if="grouped_line['display_type'] == 'line_subsection'">
                            <t t-set="current_subsection" t-value="grouped_line"/>
                        </t>
                        <t t-set="line_padding" t-value="4 if current_subsection and grouped_line['display_type'] != 'line_subsection' else                                                             3 if current_section and grouped_line['display_type'] != 'line_section' else                                                             0"/>
                        <t t-set="padding_class" t-value="line_padding and ('px-' + str(line_padding)) or ''"/>
                    </t>
                    <tr t-att-class="                                                 'fw-bolder o_line_section' if grouped_line.get('display_type') == 'line_section' and not hide_details else                                                 'fw-bold o_line_subsection' if grouped_line.get('display_type') == 'line_subsection' and not hide_details else                                                 ''">
                        <!-- Para ordenadores -->
                        <td name="account_invoice_line_name_grouped_desktop" t-att-colspan="line_colspan" t-attf-class="{{'d-none d-md-table-cell' if report_type == 'html' else ''}} {{padding_class}}">
                            <span t-out="grouped_line['name']" t-options="{'widget': 'text'}">Group 1</span>
                        </td>
                        <!-- Para movil -->
                        <td name="account_invoice_line_name_grouped" t-att-colspan="2 if grouped_line['display_type'] != 'product' else 1" t-attf-class="{{'d-md-none d-table-cell' if report_type == 'html' else 'd-none'}} {{padding_class}}">
                            <span t-out="grouped_line['name']" t-options="{'widget': 'text'}">Group 1</span>
                        </td>
                        <td name="td_quantity_grouped" colspan="1" t-if="hide_details or (hide_prices and grouped_line['display_type'] not in ('line_section', 'line_subsection', 'line_note'))" class="o_td_quantity text-end">
                            <!--  Si la línea está oculta en la composición, mostrar siempre 1 unidad -->
                            <div t-if="hide_details">
                                <span>1.00</span>
                                <span groups="uom.group_uom">Units</span>
                            </div>
                            <!-- De lo contrario, muestra la cantidad de la línea -->
                            <div t-if="hide_prices and grouped_line['display_type'] not in ('line_section', 'line_subsection')">
                                <span t-out="grouped_line['quantity']" t-options="{'widget': 'float', 'decimal_precision': 'Product Unit'}">1.00</span>
                                <t t-if="grouped_line.get('line_uom')">
                                    <span t-out="grouped_line['line_uom'].display_name" groups="uom.group_uom">Unit</span>
                                    <span t-if="grouped_line.get('product_uom') != grouped_line['line_uom']" groups="uom.group_uom" class="text-muted small">
                                        <br/>
                                        <t t-set="product_uom_qty" t-value="grouped_line['line_uom']._compute_quantity(grouped_line['quantity'], grouped_line['product_uom'])"/>
                                        <span t-out="product_uom_qty" t-options="{'widget': 'float', 'decimal_precision': 'Product Unit'}" data-oe-demo="3.00"/> <span t-out="grouped_line['product_uom'].display_name" data-oe-demo="units"/>
                                    </span>
                                </t>
                            </div>
                        </td>
                        <td name="td_price_unit_grouped" t-if="hide_details or grouped_line['display_type'] not in ('line_section', 'line_subsection')" colspan="1" t-attf-class="text-end {{ 'd-none d-md-table-cell' if report_type == 'html' else '' }}">
                            <span t-if="not hide_prices and not hide_details" t-out="grouped_line['price_subtotal']" t-options="{'widget': 'float', 'decimal_precision': 'Product Price'}" class="text-nowrap">9.00</span>
                        </td>
                        <td name="td_discount_grouped" t-if="display_discount and (hide_details or grouped_line['display_type'] == 'product')" colspan="1" t-attf-class="text-end {{ 'd-none d-md-table-cell' if report_type == 'html' else '' }}"/>
                        <td name="td_taxes_grouped" colspan="1" t-if="display_taxes and (hide_details or grouped_line['display_type'] == 'product' or grouped_line.get('taxes'))" t-attf-class="text-end {{'d-none d-md-table-cell' if report_type == 'html' else ''}}">
                            <span t-if="grouped_line.get('taxes')" t-out="','.join(grouped_line['taxes'])" id="grouped_line_tax_ids">Tax 15%</span>
                        </td>
                        <td name="td_subtotal_grouped" colspan="1" class="text-end o_price_total">
                            <span t-if="(grouped_line.get('display_type') in ('line_section', 'line_subsection') or hide_details) and o.company_price_include == 'tax_excluded'" class="text-nowrap" t-out="grouped_line['price_subtotal']" t-options="{&quot;widget&quot;: &quot;monetary&quot;, &quot;display_currency&quot;: o.currency_id}">27.00</span>
                            <span t-if="(grouped_line.get('display_type') in ('line_section', 'line_subsection') or hide_details) and o.company_price_include == 'tax_included'" class="text-nowrap" t-out="grouped_line['price_total']" t-options="{&quot;widget&quot;: &quot;monetary&quot;, &quot;display_currency&quot;: o.currency_id}">31.05</span>
                        </td>
                    </tr>
                </t>
            </t>
        </t>
    </tbody>
```

3. **Lógica condicional:** Debe implementar una condición (t-if) para ocultar la columna de "Descuento" en caso de que ninguna de las líneas de detalle de la factura lo contenga.

```
<t t-set="display_discount" t-value="any(line.discount for line in o.invoice_line_ids if line.display_type == 'product')"/>
```

4. **Data Binding:** Utilizar la directiva t-field para mostrar de forma limpia campos como el número de factura (doc.name), fecha de emisión (doc.date) y el total neto (doc.amount_total).

```
<!-- Nombre del documento: número de factura enlazado con t-field -->
    <span t-if="doc.name and doc.name != '/'" t-field="doc.name">INV/2023/0001</span>

<!-- Fecha del documento mostrada con t-field para formateo automático -->
    <div t-field="doc.invoice_date">2023-09-12</div>

<!-- Total neto del documento mostrado como moneda -->
    <span t-field="doc.amount_total" t-options="{'widget': 'monetary', 'display_currency': doc.currency_id}"/>
```

### Entregable 2: Interoperabilidad de Datos (Extracción JSON/XML)

1. **invoice\_ubl.xml:** Un fragmento de factura electrónica simplificado que cumpla con las especificaciones del estándar internacional **UBL (Universal Business Language)**, incluyendo obligatoriamente los namespaces de componentes agregados (cac) y componentes básicos (cbc), así como el ID de personalización europeo compatible con la red PEPPOL.

### Entregable 3: Manual de Explotación bajo Norma ISO/IEC/IEEE 26514

1. **Introducción y Arquitectura:** Descripción técnica de los módulos activados en el ERP, indicando la topología lógica (ej. despliegue mediante Docker Compose).  
2. **Guía de Instalación y Reinstalación:** Pasos detallados para levantar el entorno desde cero, indicando variables de entorno necesarias y dependencias del SGBD.  
3. **Seguridad y Control de Acceso:** Configuración de roles (administrador, contable, comercial), políticas de contraseñas y privilegios sobre la información.  
4. **Procedimiento de Backup y Restauración:** Detalle del comando para respaldar la base de datos relacional y los almacenes de datos asociados.  
5. **Flujo Operativo de Facturación e Informes:** Explicación paso a paso de cómo un usuario genera una factura en la interfaz y cómo el sistema renderiza el informe final a PDF (explicando de manera didáctica el pipeline HTML \-\> wkhtmltopdf \-\> PDF).

