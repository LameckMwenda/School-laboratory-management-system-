# School-laboratory-management-system-
School laboratory manager 
# myproject/settings.py
# Add 'labsystem' to the INSTALLED_APPS list
# Add 'rest_framework' to the INSTALLED_APPS list

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'labsystem', # Your new app
    'rest_framework',
    'corsheaders', # We need this for cross-origin requests
]

# Add CORS middleware to the MIDDLEWARE list
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware', # Add this line
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Set allowed origins for CORS
CORS_ALLOW_ALL_ORIGINS = True # In a real app, you would specify a list of allowed origins

# -----------------------------------------------------------

# myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('labsystem.urls')),
]

# -----------------------------------------------------------

# labsystem/models.py
from django.db import models

class School(models.Model):
    """Model for a school in the system."""
    school_code = models.CharField(max_length=10, unique=True)
    school_name = models.CharField(max_length=200)

    def __str__(self):
        return self.school_name

class Staff(models.Model):
    """Model for a staff member."""
    user_code = models.CharField(max_length=10, unique=True)
    name = models.CharField(max_length=200)
    role = models.CharField(max_length=50) # e.g., 'School Admin', 'Lab Technician', 'Teacher'
    school = models.ForeignKey(School, on_delete=models.CASCADE, null=True, blank=True)

    def __str__(self):
        return self.name

class InventoryItem(models.Model):
    """Model for an inventory item."""
    name = models.CharField(max_length=200)
    category = models.CharField(max_length=100)
    quantity = models.IntegerField(default=0)
    min_stock = models.IntegerField(default=0)
    school = models.ForeignKey(School, on_delete=models.CASCADE)

    def __str__(self):
        return self.name

# -----------------------------------------------------------

# labsystem/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('login/', views.login_view, name='login'),
    path('schools/', views.SchoolListCreate.as_view(), name='schools_list_create'),
    path('staff/', views.StaffListCreate.as_view(), name='staff_list_create'),
    path('inventory/', views.InventoryListCreate.as_view(), name='inventory_list_create'),
    path('data/', views.InitialDataView.as_view(), name='initial_data'),
]

# -----------------------------------------------------------

# labsystem/views.py
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.shortcuts import get_object_or_404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.decorators import api_view
from .models import School, Staff, InventoryItem
from .serializers import SchoolSerializer, StaffSerializer, InventoryItemSerializer

# Initial data setup - this will run once on the first server start
# In a real app, this would be handled by fixtures or migrations
def initial_data_setup():
    if not School.objects.exists():
        school_a = School.objects.create(school_code='SCHOOLA', school_name='Greenwood High')
        school_b = School.objects.create(school_code='SCHOOLB', school_name='Springfield Academy')
        Staff.objects.create(user_code='ADMIN001', name='Overall Admin', role='Overall Admin')
        Staff.objects.create(user_code='S-001', name='Jane Doe', role='School Admin', school=school_a)
        Staff.objects.create(user_code='LT-001', name='John Smith', role='Lab Technician', school=school_a)
        Staff.objects.create(user_code='S-002', name='Alice Williams', role='School Admin', school=school_b)
        Staff.objects.create(user_code='LT-002', name='Bob Johnson', role='Lab Technician', school=school_b)
        InventoryItem.objects.create(name='Beaker 250ml', category='Glassware', quantity=50, min_stock=10, school=school_a)
        InventoryItem.objects.create(name='Sulfuric Acid', category='Chemicals', quantity=5, min_stock=2, school=school_a)
        InventoryItem.objects.create(name='Test Tube Rack', category='Equipment', quantity=20, min_stock=5, school=school_b)
        InventoryItem.objects.create(name='Bunsen Burner', category='Equipment', quantity=10, min_stock=5, school=school_b)

    # Note: Call this function only once. For a persistent solution, use Django's data migrations.
    # We will call it in `InitialDataView` to ensure data exists if the database is empty.

@api_view(['POST'])
def login_view(request):
    """API endpoint to handle user login."""
    school_code = request.data.get('schoolCode')
    user_code = request.data.get('userCode')

    if user_code == 'ADMIN001' and school_code == 'ADMIN001':
        return Response({'userCode': user_code, 'schoolCode': None, 'role': 'Overall Admin'})

    try:
        staff_member = Staff.objects.get(user_code=user_code, school__school_code=school_code)
        return Response({
            'userCode': staff_member.user_code,
            'schoolCode': staff_member.school.school_code,
            'role': staff_member.role
        })
    except Staff.DoesNotExist:
        return Response({'message': 'Invalid School Code or User Code'}, status=401)

class InitialDataView(APIView):
    """
    API endpoint to provide all initial data.
    Note: For a real app, this would be split into more specific endpoints.
    """
    def get(self, request):
        initial_data_setup() # Ensure initial data exists for the prototype
        schools = SchoolSerializer(School.objects.all(), many=True).data
        staff_data = {}
        inventory_data = {}

        all_staff = Staff.objects.exclude(role='Overall Admin')
        for member in all_staff:
            if member.school:
                school_code = member.school.school_code
                if school_code not in staff_data:
                    staff_data[school_code] = []
                staff_data[school_code].append(StaffSerializer(member).data)

        all_inventory = InventoryItem.objects.all()
        for item in all_inventory:
            school_code = item.school.school_code
            if school_code not in inventory_data:
                inventory_data[school_code] = []
            inventory_data[school_code].append(InventoryItemSerializer(item).data)

        return Response({
            'schools': schools,
            'staff': staff_data,
            'inventory': inventory_data
        })

class SchoolListCreate(APIView):
    """
    API endpoint for adding a new school.
    """
    def post(self, request):
        if request.data.get('userRole') != 'Overall Admin':
            return Response({'message': 'Access denied'}, status=403)
        
        serializer = SchoolSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)

class StaffListCreate(APIView):
    """
    API endpoint for adding a new staff member.
    """
    def post(self, request):
        if request.data.get('userRole') != 'School Admin':
            return Response({'message': 'Access denied'}, status=403)
        
        school_code = request.data.get('schoolCode')
        school = get_object_or_404(School, school_code=school_code)
        
        serializer = StaffSerializer(data={
            'user_code': request.data.get('userCode'),
            'name': request.data.get('name'),
            'role': request.data.get('role'),
            'school': school.id
        })
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)

class InventoryListCreate(APIView):
    """
    API endpoint for adding a new inventory item.
    """
    def post(self, request):
        user_role = request.data.get('userRole')
        if user_role not in ['School Admin', 'Lab Technician']:
            return Response({'message': 'Access denied'}, status=403)
        
        school_code = request.data.get('schoolCode')
        school = get_object_or_404(School, school_code=school_code)

        serializer = InventoryItemSerializer(data={
            'name': request.data.get('name'),
            'category': request.data.get('category'),
            'quantity': request.data.get('quantity'),
            'min_stock': request.data.get('minStock'),
            'school': school.id
        })
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)

# -----------------------------------------------------------

# labsystem/serializers.py
from rest_framework import serializers
from .models import School, Staff, InventoryItem

class SchoolSerializer(serializers.ModelSerializer):
    class Meta:
        model = School
        fields = '__all__'

class StaffSerializer(serializers.ModelSerializer):
    class Meta:
        model = Staff
        fields = '__all__'

class InventoryItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = InventoryItem
        fields = '__all__'
