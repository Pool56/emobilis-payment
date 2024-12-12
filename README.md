# emobilis-payment
This is a program written in Django which facilitates payment

from django.db import models
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
import requests

# 1. Models
class Payment(models.Model):
    phone_number = models.CharField(max_length=15)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    transaction_id = models.CharField(max_length=100, null=True, blank=True)
    status = models.CharField(max_length=50, default="Pending")
    created_at = models.DateTimeField(auto_now_add=True)

# 2. USSD Logic
@csrf_exempt
@api_view(['POST'])
def ussd_interface(request):
    session_id = request.data.get('sessionId')
    service_code = request.data.get('serviceCode')
    phone_number = request.data.get('phoneNumber')
    text = request.data.get('text', '')

    # Parse USSD input
    user_input = text.split('*')

    if text == '':
        # Main menu
        response = "CON Welcome to MPESA Payment Service\n"
        response += "1. Create Payment\n"
        response += "2. View Payment\n"
        response += "3. Update Payment\n"
        response += "4. Delete Payment"
    elif text == '1':
        response = "CON Enter Amount:"
    elif len(user_input) == 2 and user_input[0] == '1':
        # Create Payment
        amount = user_input[1]
        payment = Payment.objects.create(phone_number=phone_number, amount=amount)
        response = f"END Payment of KES {amount} created with ID {payment.id}."
    elif text == '2':
        # View Payments
        payments = Payment.objects.filter(phone_number=phone_number)
        if payments.exists():
            response = "END Your Payments:\n"
            for p in payments:
                response += f"ID: {p.id}, Amount: {p.amount}, Status: {p.status}\n"
        else:
            response = "END No payments found."
    elif text == '3':
        response = "CON Enter Payment ID to Update:"
    elif len(user_input) == 2 and user_input[0] == '3':
        payment_id = user_input[1]
        try:
            payment = Payment.objects.get(id=payment_id, phone_number=phone_number)
            payment.status = "Updated"
            payment.save()
            response = f"END Payment ID {payment_id} updated successfully."
        except Payment.DoesNotExist:
            response = "END Payment not found."
    elif text == '4':
        response = "CON Enter Payment ID to Delete:"
    elif len(user_input) == 2 and user_input[0] == '4':
        payment_id = user_input[1]
        try:
            payment = Payment.objects.get(id=payment_id, phone_number=phone_number)
            payment.delete()
            response = f"END Payment ID {payment_id} deleted successfully."
        except Payment.DoesNotExist:
            response = "END Payment not found."
    else:
        response = "END Invalid input."

    return JsonResponse({'text': response})

# 3. MPESA Integration (Simplified Example)
def initiate_mpesa_payment(phone_number, amount):
    url = "https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest"
    headers = {
        "Authorization": "Bearer ACCESS_TOKEN",
        "Content-Type": "application/json"
    }
    payload = {
        "BusinessShortCode": "174379",
        "Password": "ENCODED_PASSWORD",
        "Timestamp": "TIMESTAMP",
        "TransactionType": "CustomerPayBillOnline",
        "Amount": amount,
        "PartyA": phone_number,
        "PartyB": "174379",
        "PhoneNumber": phone_number,
        "CallBackURL": "https://yourdomain.com/callback",
        "AccountReference": "Payment",
        "TransactionDesc": "Payment Description"
    }
    response = requests.post(url, json=payload, headers=headers)
    return response.json()
