<?php
/**
 * Plugin Name: WooCommerce DNI/NIF/NIE for Invoices
 * Description: Agrega automáticamente un campo DNI/NIF/NIE en el checkout y lo incluye en la factura PDF de WooCommerce.
 * Version: 1.8.1
 * Author: David Revilla
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

function verificar_plugins_woocommerce_pdf() {
    // Verifica que WooCommerce y WooCommerce PDF Invoices & Packing Slips estén activos
    if ( ! class_exists( 'WooCommerce' ) || ! class_exists( 'WPO_WCPDF' ) ) {
        return;
    }
    
    // Agregar el campo DNI/NIF/NIE en el checkout (debajo del campo "nombre de la empresa")
    add_filter( 'woocommerce_checkout_fields', function ( $fields ) {
        $fields['billing']['billing_dni'] = array(
            'label'       => __( 'DNI / NIF / NIE', 'woocommerce' ),
            'placeholder' => __( 'Introduce tu número de identificación', 'woocommerce' ),
            'required'    => false,
            'clear'       => false,
            'type'        => 'text',
            'class'       => array( 'form-row-wide' ),
            'priority'    => 25, // Se posiciona debajo del campo de empresa
        );
        return $fields;
    } );
    
    // Guardar el DNI en los metadatos del pedido de forma segura
    add_action( 'woocommerce_checkout_update_order_meta', function ( $order_id ) {
        if ( ! empty( $_POST['billing_dni'] ) ) {
            update_post_meta( $order_id, '_billing_dni', sanitize_text_field( $_POST['billing_dni'] ) );
        }
    } );
    
    // Mostrar el DNI en el área de facturación del panel de administración
    add_filter( 'woocommerce_admin_billing_fields', function ( $fields ) {
        $fields['billing_dni'] = array(
            'label' => __( 'DNI / NIF / NIE', 'woocommerce' ),
            'show'  => true,
        );
        return $fields;
    } );
    
    // Añadir el DNI (mostrado como "NIF:") a la dirección de facturación en la factura PDF
    add_filter( 'wpo_wcpdf_billing_address', 'agregar_dni_a_factura', 10, 2 );
}
add_action( 'plugins_loaded', 'verificar_plugins_woocommerce_pdf' );

/**
 * Función para agregar el DNI (como "NIF:") a la dirección de facturación en la factura PDF.
 *
 * @param string $address La dirección de facturación ya generada.
 * @param mixed  $document El objeto documento (factura), que contiene el pedido.
 * @return string La dirección de facturación modificada.
 */
function agregar_dni_a_factura( $address, $document ) {
    $order_id = 0;
    
    // Si el objeto documento es en realidad un WC_Order, se utiliza directamente.
    if ( method_exists( $document, 'get_id' ) ) {
        $order_id = $document->get_id();
    }
    // Si no, comprobamos si tiene la propiedad "order" que contenga el pedido real.
    elseif ( isset( $document->order ) && method_exists( $document->order, 'get_id' ) ) {
        $order_id = $document->order->get_id();
    }
    
    if ( $order_id ) {
        $dni = get_post_meta( $order_id, '_billing_dni', true );
        if ( ! empty( $dni ) ) {
            $address .= '<br><strong>' . esc_html( __( 'NIF:', 'woocommerce' ) ) . '</strong> ' . esc_html( $dni );
        }
    }
    
    return $address;
}
?>
