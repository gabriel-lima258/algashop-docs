# Cartões de teste

Os cartões de teste do cliente 6e148bd5-47f6-4022-b9da-07cfaa294f7a já estão cadastrados no Billing. Cada um está ligado a um cartão do FastPay com resultado fixo, e o checkout deve ser feito autenticado como esse cliente:

creditCardId no Billing	Cartão	Resultado no FastPay
- 0199deda-9486-7d27-b40c-761f561a24d8	Visa final 1022	Pago na captura
- 0199e2dd-9434-7a09-9241-f23536267484	Master final 4545	Em processamento na captura, pago depois pelo webhook
- 019d4920-3814-7629-ac43-0f83f7888f0d	Visa final 1030	Recusado